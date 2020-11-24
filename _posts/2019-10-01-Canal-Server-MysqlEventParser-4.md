---
layout: post
title: 'Canal Server源码之六(MysqlEventParser-4)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). DirectLogFetcher
```
public void seek(
        String binlogfilename, 
        Long binlogPosition, 
        String gtid, 
        SinkFunction func) throws IOException {

    updateSettings();
    loadBinlogChecksum();
    // 向MySQL发起binlogdump请求
    sendBinlogDump(binlogfilename, binlogPosition);

    // 创建:DirectLogFetcher
    // connector.getReceiveBufferSize() = 16384
    DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
    // 设置Channel
    fetcher.start(connector.getChannel());

    // 创建:LogDecoder
    LogDecoder decoder = new LogDecoder();
    decoder.handle(LogEvent.ROTATE_EVENT);
    decoder.handle(LogEvent.FORMAT_DESCRIPTION_EVENT);
    decoder.handle(LogEvent.QUERY_EVENT);
    decoder.handle(LogEvent.XID_EVENT);

    // 创建上下文
    LogContext context = new LogContext();
    // 若entry position存在gtid，则使用传入的gtid作为gtidSet
    // 拼接的标准,否则同时开启gtid和tsdb时，会导致丢失gtid
    // 而当源端数据库gtid 有purged时会有如下类似报错
    // 'errno = 1236, sqlstate = HY000 errmsg = The slave is connecting
    // using CHANGE MASTER TO MASTER_AUTO_POSITION = 1 ...
    if (StringUtils.isNotEmpty(gtid)) {
        decoder.handle(LogEvent.GTID_LOG_EVENT);
        context.setGtidSet(MysqlGTIDSet.parse(gtid));
    }
    
    // **************************************************************************
    // 2. new FormatDescriptionLogEvent
    // **************************************************************************
    context.setFormatDescription(new FormatDescriptionLogEvent(4, binlogChecksum));

    //  DirectLogFetcher可以理解为缓存冲,Canal相当于自己写了一个缓冲区.
    // **************************************************************************
    // 3. DirectLogFetcher.fetch
    // **************************************************************************
    while (fetcher.fetch()) { // fetch是从Channel中读取第一个报文
        // fetcher.limit() = 47
        // 统计下接受了47个字节
        accumulateReceivedBytes(fetcher.limit());

        LogEvent event = null;
        // *****************************************************************
        // 5.调用LogDecoder.decode()进行解码
        // *****************************************************************
        event = decoder.decode(fetcher, context);

        if (event == null) { 
            throw new CanalParseException("parse failed");
        }

        // *********************************************************************
        // 8.调用回调函数(func),如果回调函数返回:false,则跳出循环
        // MysqlEventParser.findAsPerTimestampInSpecificLogFile创建的内部匿名函数 
        // *********************************************************************
        if (!func.sink(event)) {
            break;
        }
    }
}// end seek
```
### (2). FormatDescriptionLogEvent
```
public final class FormatDescriptionLogEvent 
             extends StartLogEventV3 {

    // 2.1 构造器
    public FormatDescriptionLogEvent(final int binlogVersion, int binlogChecksum){
        this(binlogVersion);
        // binlogChecksum = 1
        this.header.checksumAlg = binlogChecksum;
    } //end FormatDescriptionLogEvent
}// end FormatDescriptionLogEvent
```
### (3). DirectLogFetcher.fetch
```

public boolean fetch() throws IOException {
    try {
        // 3.1 fetch0读取package里的header
        // Fetching packet header from input.
        if (!fetch0(0, NET_HEADER_SIZE)) {
            logger.warn("Reached end of input stream while fetching header");
            return false;
        }

        // 获取package的第一个字节
        // Fetching the first packet(may a multi-packet).
        // netlen = 48
        int netlen = getUint24(PACKET_LEN_OFFSET);
        // netnum = 1
        int netnum = getUint8(PACKET_SEQ_OFFSET);

        // NET_HEADER_SIZE = 4
        // fetch0(4,48)
        // 从输入流中读取:52个字节到buffer(协议体的内容)
        if (!fetch0(NET_HEADER_SIZE, netlen)) { // false
            logger.warn("Reached end of input stream: packet #" + netnum + ", len = " + netlen);
            return false;
        }// end if

        // 读取协议头里,定义协议体的大小
        // Detecting error code.
        // mark = 0
        final int mark = getUint8(NET_HEADER_SIZE);

        if (mark != 0) { // false
            if (mark == 255) // error from master
            {
                // Indicates an error, for example trying to fetch from
                // wrong
                // binlog position.
                position = NET_HEADER_SIZE + 1;
                final int errno = getInt16();
                String sqlstate = forward(1).getFixString(SQLSTATE_LENGTH);
                String errmsg = getFixString(limit - position);
                throw new IOException("Received error packet:" + " errno = " + errno + ", sqlstate = " + sqlstate
                                        + " errmsg = " + errmsg);
            } else if (mark == 254) {
                // Indicates end of stream. It's not clear when this would
                // be sent.
                logger.warn("Received EOF packet from server, apparent"
                            + " master disconnected. It's may be duplicate slaveId , check instance config");
                return false;
            } else {
                // Should not happen.
                throw new IOException("Unexpected response " + mark + " while fetching binlog: packet #" + netnum
                                        + ", len = " + netlen);
            }
        } // end else

        // if mysql is in semi mode
        if (issemi) {   //false
            // parse semi mark
            int semimark = getUint8(NET_HEADER_SIZE + 1);
            int semival = getUint8(NET_HEADER_SIZE + 2);
            this.semival = semival;
        }

        // The first packet is a multi-packet, concatenate the packets.
        // netlen = 48
        // MAX_PACKET_LENGTH = 16777215
        while (netlen == MAX_PACKET_LENGTH) { // false
            if (!fetch0(0, NET_HEADER_SIZE)) {
                logger.warn("Reached end of input stream while fetching header");
                return false;
            }

            netlen = getUint24(PACKET_LEN_OFFSET);
            netnum = getUint8(PACKET_SEQ_OFFSET);
            if (!fetch0(limit, netlen)) {
                logger.warn("Reached end of input stream: packet #" + netnum + ", len = " + netlen);
                return false;
            }
        }// end while

        // Preparing buffer variables to decoding.
        if (issemi) { // false
            origin = NET_HEADER_SIZE + 3;
        } else { //true
            // origin = 4 + 1;
            origin = NET_HEADER_SIZE + 1;
        }
        // position = 5
        position = origin;
        // limit = 52 - 5
        // limit = 47
        limit -= origin;
        return true;
    } catch (SocketTimeoutException e) {
        close(); /* Do cleanup */
        logger.error("Socket timeout expired, closing connection", e);
        throw e;
    } catch (InterruptedIOException e) {
        close(); /* Do cleanup */
        logger.info("I/O interrupted while reading from client socket", e);
        throw e;
    } catch (ClosedByInterruptException e) {
        close(); /* Do cleanup */
        logger.info("I/O interrupted while reading from client socket", e);
        throw e;
    } catch (IOException e) {
        close(); /* Do cleanup */
        logger.error("I/O error while reading from client socket", e);
        throw e;
    }
} //end fetch


// 3.2 fetch0
private final boolean fetch0(final int off, final int len) throws IOException {
    // off = 0len
    // len = 4

    // 3.3 ensureCapacity
    ensureCapacity(off + len);


    // byte[] read = channel.read(len, READ_TIMEOUT_MILLISECONDS);
    // System.arraycopy(read, 0, this.buffer, off, len);

    // *******************************************************************
    // 4.从通道中读取指定字节的数据到缓存冲区里(channel = BioSocketChannel)
    // *******************************************************************
    channel.read(buffer, off, len, READ_TIMEOUT_MILLISECONDS);
    
    if (limit < off + len) { // true
        // 设置已经写到的最大位置为:4
        limit = off + len;
    }
    return true;
} //end fetch0


// 3.4 ensureCapacity(4)
protected final void ensureCapacity(final int minCapacity) {
    // minCapacity = 4
    // oldCapacity = 16384
    final int oldCapacity = buffer.length;

    // 如果要申请的空间 > 总的空间,则需要扩容(Arrays.copyOf)
    if (minCapacity > oldCapacity) { // false
        int newCapacity = (int) (oldCapacity * factor);
        if (newCapacity < minCapacity) newCapacity = minCapacity;

        buffer = Arrays.copyOf(buffer, newCapacity);
    }
}// end ensureCapacity

```
### (4). BioSocketChannel.read
```
public class BioSocketChannel implements SocketChannel {
    private Socket       socket;
    private InputStream  input;
    private OutputStream output;

    
    public void read(byte[] data, int off, int len, int timeout) throws IOException {
        // data = []
        // off = 0
        // len = 4
        // timeout = 25000
        InputStream input = this.input;
        int accTimeout = 0;
        if (input == null) {
            throw new SocketException("Socket already closed.");
        }

        int n = 0;
        while (n < len && accTimeout < timeout) {
            try {
                // 从通道中读取数据
                int read = input.read(data, off + n, len - n);
                if (read > -1) {
                    n += read;
                } else {
                    throw new IOException("EOF encountered.");
                }
            } catch (SocketTimeoutException te) {
                if (Thread.interrupted()) {
                    throw new ClosedByInterruptException();
                }
                accTimeout += SO_TIMEOUT;
            }
        }

        if (n < len && accTimeout >= timeout) { 
            // 没有读到指定数量的数据,或者已经超时,则抛出异常
            throw new SocketTimeoutException("Timeout occurred, failed to read total " + len + " bytes in " + timeout
                                             + " milliseconds, actual read only " + n + " bytes");
        }
    }// end read
}
```
### (5). LogDecoder.decode
```
public LogEvent decode(LogBuffer buffer, LogContext context) throws IOException {
    // limit = 47
    // 从缓冲区中,获得报文体的内容
    final int limit = buffer.limit();

    // FormatDescriptionLogEvent.LOG_EVENT_HEADER_LEN = 19
    // limit = 47
    if (limit >= FormatDescriptionLogEvent.LOG_EVENT_HEADER_LEN) { // true
        
        // ********************************************************************
        // 读取buffer中的内容,填充到LogHeader模型中
        // ********************************************************************
        LogHeader header = new LogHeader(buffer, context.getFormatDescription());
        // len = 47
        final int len = header.getEventLen();
        if (limit >= len) {  //true
            LogEvent event;

             // 在MysqlConnection.seek方法中初始了:LogDecoder,同时添加了handle
            // handleSet = {2, 4, 15, 16}
            /* Checking binary-log's header */
            if (handleSet.get(header.getType())) { // true
                //设置buffer.limit(47)
                buffer.limit(len);
                try {
                    /* Decoding binary-log to event */
                    // ************************************************************
                    // 7.LogEvent.decode
                    // ************************************************************
                    event = decode(buffer, header, context);
                } catch (IOException e) {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Decoding " + LogEvent.getTypeName(header.getType()) + " failed from: "
                                    + context.getLogPosition(), e);
                    }
                    throw e;
                } finally {
                    // 重置limit(47)
                    buffer.limit(limit); /* Restore limit */
                }
            } else {
                /* Ignore unsupported binary-log. */
                event = new UnknownLogEvent(header);
            }

            if (event != null) { // true
                // set logFileName
                // 在事件头里设置logfilename
                event.getHeader().setLogFileName(context.getLogPosition().getFileName());
                // buffer.semival = 0
                event.setSemival(buffer.semival);
            }

            // 重置缓冲区中的标识
            /* consume this binary-log. */
            buffer.consume(len);
            return event;
        }
    }

    /* Rewind buffer's position to 0. */
    buffer.rewind();
    return null;
}// end decode
```
### (6). LogHeader
```
public LogHeader(LogBuffer buffer, FormatDescriptionLogEvent descriptionEvent){
    // when = 0
    when = buffer.getUint32();
    // type = 4
    type = buffer.getUint8(); // LogEvent.EVENT_TYPE_OFFSET;
    // serverId = 1
    serverId = buffer.getUint32(); // LogEvent.SERVER_ID_OFFSET;
    // eventLen = 47
    eventLen = (int) buffer.getUint32(); // LogEvent.EVENT_LEN_OFFSET;

    if (descriptionEvent.binlogVersion == 1) { // false
        logPos = 0;
        flags = 0;
        return;
    }

    /* 4.0 or newer */
    // logPos = 0
    logPos = buffer.getUint32(); // LogEvent.LOG_POS_OFFSET
    /*
        * If the log is 4.0 (so here it can only be a 4.0 relay log read by the
        * SQL thread or a 4.0 master binlog read by the I/O thread), log_pos is
        * the beginning of the event: we transform it into the end of the
        * event, which is more useful. But how do you know that the log is 4.0:
        * you know it if description_event is version 3 *and* you are not
        * reading a Format_desc (remember that mysqlbinlog starts by assuming
        * that 5.0 logs are in 4.0 format, until it finds a Format_desc).
        */
    if (descriptionEvent.binlogVersion == 3 && type < LogEvent.FORMAT_DESCRIPTION_EVENT && logPos != 0) { // false
        /*
            * If log_pos=0, don't change it. log_pos==0 is a marker to mean
            * "don't change rli->group_master_log_pos" (see
            * inc_group_relay_log_pos()). As it is unreal log_pos, adding the
            * event len's is nonsense. For example, a fake Rotate event should
            * not have its log_pos (which is 0) changed or it will modify
            * Exec_master_log_pos in SHOW SLAVE STATUS, displaying a nonsense
            * value of (a non-zero offset which does not exist in the master's
            * binlog, so which will cause problems if the user uses this value
            * in CHANGE MASTER).
            */
        logPos += eventLen; /* purecov: inspected */
    }

    // flags = 32
    flags = buffer.getUint16(); // LogEvent.FLAGS_OFFSET

    if ((type == LogEvent.FORMAT_DESCRIPTION_EVENT) || (type == LogEvent.ROTATE_EVENT)) { //false
        /*
            * These events always have a header which stops here (i.e. their
            * header is FROZEN).
            */
        /*
            * Initialization to zero of all other Log_event members as they're
            * not specified. Currently there are no such members; in the future
            * there will be an event UID (but Format_description and Rotate
            * don't need this UID, as they are not propagated through
            * --log-slave-updates (remember the UID is used to not play a query
            * twice when you have two masters which are slaves of a 3rd
            * master). Then we are done.
            */

        if (type == LogEvent.FORMAT_DESCRIPTION_EVENT) { // false
            int commonHeaderLen = buffer.getUint8(FormatDescriptionLogEvent.LOG_EVENT_MINIMAL_HEADER_LEN
                                                    + FormatDescriptionLogEvent.ST_COMMON_HEADER_LEN_OFFSET);
            buffer.position(commonHeaderLen + FormatDescriptionLogEvent.ST_SERVER_VER_OFFSET);
            String serverVersion = buffer.getFixString(FormatDescriptionLogEvent.ST_SERVER_VER_LEN); // ST_SERVER_VER_OFFSET
            int versionSplit[] = new int[] { 0, 0, 0 };
            FormatDescriptionLogEvent.doServerVersionSplit(serverVersion, versionSplit);
            checksumAlg = LogEvent.BINLOG_CHECKSUM_ALG_UNDEF;
            if (FormatDescriptionLogEvent.versionProduct(versionSplit) >= FormatDescriptionLogEvent.checksumVersionProduct) {
                buffer.position(eventLen - LogEvent.BINLOG_CHECKSUM_LEN - LogEvent.BINLOG_CHECKSUM_ALG_DESC_LEN);
                checksumAlg = buffer.getUint8();
            }

            processCheckSum(buffer);
        }
        return;
}// end LogHeader
```
### (7). LogDecoder.decode
```
public static LogEvent decode(LogBuffer buffer, LogHeader header, LogContext context) throws IOException {
    FormatDescriptionLogEvent descriptionEvent = context.getFormatDescription();
    LogPosition logPosition = context.getLogPosition();

    // checksumAlg = 255
    int checksumAlg = LogEvent.BINLOG_CHECKSUM_ALG_UNDEF;
    // header.getType() = 4
    // LogEvent.FORMAT_DESCRIPTION_EVENT = 15
    if (header.getType() != LogEvent.FORMAT_DESCRIPTION_EVENT) { // true
        //  checksumAlg = 1
        checksumAlg = descriptionEvent.header.getChecksumAlg();
    } else {
        // 如果是format事件自己，也需要处理checksum
        checksumAlg = header.getChecksumAlg();
    }

    // LogEvent.BINLOG_CHECKSUM_ALG_OFF = 0
    // checksumAlg = 1
    // LogEvent.BINLOG_CHECKSUM_ALG_UNDEF = 255
    if (checksumAlg != LogEvent.BINLOG_CHECKSUM_ALG_OFF 
       && checksumAlg != LogEvent.BINLOG_CHECKSUM_ALG_UNDEF) {
        // remove checksum bytes
        // header.getEventLen() = 47
        // LogEvent.BINLOG_CHECKSUM_LEN = 4
        // buffer.limit(43)
        buffer.limit(header.getEventLen() - LogEvent.BINLOG_CHECKSUM_LEN);
    }

    // gtidSet = null
    GTIDSet gtidSet = context.getGtidSet();
    // gtidLogEvent = null
    GtidLogEvent gtidLogEvent = context.getGtidLogEvent();
    // 判断协议头上的类型属于什么事件?
    // header.getType() = 4
    // 4 = LogEvent.ROTATE_EVENT
    switch (header.getType()) {
        case LogEvent.QUERY_EVENT: {
            QueryLogEvent event = new QueryLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.XID_EVENT: {
            XidLogEvent event = new XidLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.TABLE_MAP_EVENT: {
            TableMapLogEvent mapEvent = new TableMapLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            context.putTable(mapEvent);
            return mapEvent;
        }
        case LogEvent.WRITE_ROWS_EVENT_V1: {
            RowsLogEvent event = new WriteRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.UPDATE_ROWS_EVENT_V1: {
            RowsLogEvent event = new UpdateRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.DELETE_ROWS_EVENT_V1: {
            RowsLogEvent event = new DeleteRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.ROTATE_EVENT: {
            // 读取buffer内容,并填充到模型:RotateLogEvent中
            RotateLogEvent event = new RotateLogEvent(header, buffer, descriptionEvent);

            // ****************************************************************
            // 结论:LogContext里面记录着:LogPosition/GTIDSet
            // 第一次初始化为空,每次拉取事件,都会对这些内容进行改变.
            // ****************************************************************
            /* updating position in context */
            // event.getFilename() = mysql-bin.000021
            // event.getPosition() = 4
            logPosition = new LogPosition(event.getFilename(), event.getPosition());
            context.setLogPosition(logPosition);
            return event;
        }
        case LogEvent.LOAD_EVENT:
        case LogEvent.NEW_LOAD_EVENT: {
            LoadLogEvent event = new LoadLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.SLAVE_EVENT: /* can never happen (unused event) */
        {
            if (logger.isWarnEnabled()) logger.warn("Skipping unsupported SLAVE_EVENT from: "
                                                    + context.getLogPosition());
            break;
        }
        case LogEvent.CREATE_FILE_EVENT: {
            CreateFileLogEvent event = new CreateFileLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.APPEND_BLOCK_EVENT: {
            AppendBlockLogEvent event = new AppendBlockLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.DELETE_FILE_EVENT: {
            DeleteFileLogEvent event = new DeleteFileLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.EXEC_LOAD_EVENT: {
            ExecuteLoadLogEvent event = new ExecuteLoadLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.START_EVENT_V3: {
            /* This is sent only by MySQL <=4.x */
            StartLogEventV3 event = new StartLogEventV3(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.STOP_EVENT: {
            StopLogEvent event = new StopLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.INTVAR_EVENT: {
            IntvarLogEvent event = new IntvarLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.RAND_EVENT: {
            RandLogEvent event = new RandLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.USER_VAR_EVENT: {
            UserVarLogEvent event = new UserVarLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.FORMAT_DESCRIPTION_EVENT: {
            descriptionEvent = new FormatDescriptionLogEvent(header, buffer, descriptionEvent);
            context.setFormatDescription(descriptionEvent);
            return descriptionEvent;
        }
        case LogEvent.PRE_GA_WRITE_ROWS_EVENT: {
            if (logger.isWarnEnabled()) logger.warn("Skipping unsupported PRE_GA_WRITE_ROWS_EVENT from: "
                                                    + context.getLogPosition());
            // ev = new Write_rows_log_event_old(buf, event_len,
            // description_event);
            break;
        }
        case LogEvent.PRE_GA_UPDATE_ROWS_EVENT: {
            if (logger.isWarnEnabled()) logger.warn("Skipping unsupported PRE_GA_UPDATE_ROWS_EVENT from: "
                                                    + context.getLogPosition());
            // ev = new Update_rows_log_event_old(buf, event_len,
            // description_event);
            break;
        }
        case LogEvent.PRE_GA_DELETE_ROWS_EVENT: {
            if (logger.isWarnEnabled()) logger.warn("Skipping unsupported PRE_GA_DELETE_ROWS_EVENT from: "
                                                    + context.getLogPosition());
            // ev = new Delete_rows_log_event_old(buf, event_len,
            // description_event);
            break;
        }
        case LogEvent.BEGIN_LOAD_QUERY_EVENT: {
            BeginLoadQueryLogEvent event = new BeginLoadQueryLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.EXECUTE_LOAD_QUERY_EVENT: {
            ExecuteLoadQueryLogEvent event = new ExecuteLoadQueryLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.INCIDENT_EVENT: {
            IncidentLogEvent event = new IncidentLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.HEARTBEAT_LOG_EVENT: {
            HeartbeatLogEvent event = new HeartbeatLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.IGNORABLE_LOG_EVENT: {
            IgnorableLogEvent event = new IgnorableLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.ROWS_QUERY_LOG_EVENT: {
            RowsQueryLogEvent event = new RowsQueryLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.WRITE_ROWS_EVENT: {
            RowsLogEvent event = new WriteRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.UPDATE_ROWS_EVENT: {
            RowsLogEvent event = new UpdateRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.DELETE_ROWS_EVENT: {
            RowsLogEvent event = new DeleteRowsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.PARTIAL_UPDATE_ROWS_EVENT: {
            RowsLogEvent event = new UpdateRowsLogEvent(header, buffer, descriptionEvent, true);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            event.fillTable(context);
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.GTID_LOG_EVENT:
        case LogEvent.ANONYMOUS_GTID_LOG_EVENT: {
            GtidLogEvent event = new GtidLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            if (gtidSet != null) {
                gtidSet.update(event.getGtidStr());
                // update latest gtid
                header.putGtid(gtidSet, event);
            }
            // update current gtid event to context
            context.setGtidLogEvent(event);
            return event;
        }
        case LogEvent.PREVIOUS_GTIDS_LOG_EVENT: {
            PreviousGtidsLogEvent event = new PreviousGtidsLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.TRANSACTION_CONTEXT_EVENT: {
            TransactionContextLogEvent event = new TransactionContextLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.VIEW_CHANGE_EVENT: {
            ViewChangeEvent event = new ViewChangeEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.XA_PREPARE_LOG_EVENT: {
            XaPrepareLogEvent event = new XaPrepareLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.ANNOTATE_ROWS_EVENT: {
            AnnotateRowsEvent event = new AnnotateRowsEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            header.putGtid(context.getGtidSet(), gtidLogEvent);
            return event;
        }
        case LogEvent.BINLOG_CHECKPOINT_EVENT: {
            BinlogCheckPointLogEvent event = new BinlogCheckPointLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.GTID_EVENT: {
            MariaGtidLogEvent event = new MariaGtidLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.GTID_LIST_EVENT: {
            MariaGtidListLogEvent event = new MariaGtidListLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        case LogEvent.START_ENCRYPTION_EVENT: {
            StartEncryptionLogEvent event = new StartEncryptionLogEvent(header, buffer, descriptionEvent);
            /* updating position in context */
            logPosition.position = header.getLogPos();
            return event;
        }
        default:
            /*
                * Create an object of Ignorable_log_event for unrecognized
                * sub-class. So that SLAVE SQL THREAD will only update the
                * position and continue.
                */
            if ((buffer.getUint16(LogEvent.FLAGS_OFFSET) & LogEvent.LOG_EVENT_IGNORABLE_F) > 0) {
                IgnorableLogEvent event = new IgnorableLogEvent(header, buffer, descriptionEvent);
                /* updating position in context */
                logPosition.position = header.getLogPos();
                return event;
            } else {
                if (logger.isWarnEnabled()) {
                    logger.warn("Skipping unrecognized binlog event " + LogEvent.getTypeName(header.getType())
                                + " from: " + context.getLogPosition());
                }
            }
    }

    /* updating position in context */
    logPosition.position = header.getLogPos();
    /* Unknown or unsupported log event */
    return new UnknownLogEvent(header);
} //end decode
```
### (8). MysqlEventParser.findAsPerTimestampInSpecificLogFile
```
private EntryPosition findAsPerTimestampInSpecificLogFile(
    MysqlConnection mysqlConnection,
    // now()
    final Long startTimestamp,
    // EntryPosition[included=false,journalName=mysql-bin.000021,position=154,serverId=<null>,gtid=<null>,timestamp=<null>]
    final EntryPosition endPosition,
    // mysql-bin.000021
    final String searchBinlogFile,
    //true
    final Boolean justForPositionTimestamp) {

    // ...

    mysqlConnection.seek(
             searchBinlogFile, 
             4L, 
             endPosition.getGtid(), 
             // 回调函数
             new SinkFunction<LogEvent>() {
                 
            private LogPosition lastPosition;

            public boolean sink(LogEvent event) {
                EntryPosition entryPosition = null;
                try {
                    // **********************************************************
                    // 9.委托给:AbstractEventParser.parseAndProfilingIfNecessary
                    // 将事件转换成:CanalEntry.Entry
                    // **********************************************************
                    CanalEntry.Entry entry = parseAndProfilingIfNecessary(event, true);

                    // 当事件是:FormatDescriptionLogEvent会进入该代码块
                    // event.getWhen = 1606196516
                    // event.getWhen 为事件的时间点,也就是这个事件没有精确到秒
                    if (justForPositionTimestamp && 
                       logPosition.getPostion() == null && 
                       event.getWhen() > 0) { // false
                        // 初始位点
                        entryPosition = new EntryPosition(searchBinlogFile,
                            event.getLogPos() - event.getEventLen(),
                            event.getWhen() * 1000,
                            event.getServerId());
                        entryPosition.setGtid(event.getHeader().getGtidSetStr());
                        // entryPosition = EntryPosition[included=false,journalName=mysql-bin.000021,position=4,serverId=1,gtid=<null>,timestamp=1606196516000]
                        logPosition.setPostion(entryPosition);
                    }


                    // 直接用event的位点来处理,解决一个binlog文件里没有任何事件导致死循环无法退出的问题
                    // logfilename = mysql-bin.000021
                    String logfilename = event.getHeader().getLogFileName();

                    // 记录的是binlog end offest,
                    // 因为与其对比的offest是show master status里的end offest
                    // logfileoffset = 0
                    Long logfileoffset = event.getHeader().getLogPos();
                    // logposTimestamp = 0
                    Long logposTimestamp = event.getHeader().getWhen() * 1000;
                    // serverId = 1
                    Long serverId = event.getHeader().getServerId();

                    // 如果最小的一条记录都不满足条件，可直接退出
                    if (logposTimestamp >= startTimestamp) { // false
                        return false;
                    }

                    if (StringUtils.equals(endPosition.getJournalName(), logfilename)
                        && endPosition.getPosition() <= logfileoffset) { // false
                        return false;
                    }

                    // 继续控制循环往下走
                    if (entry == null) { //true
                        return true;
                    }

                    // 记录一下上一个事务结束的位置，即下一个事务的position
                    // position = current +
                    // data.length，代表该事务的下一条offest，避免多余的事务重复
                    if (CanalEntry.EntryType.TRANSACTIONEND.equals(entry.getEntryType())) {
                        entryPosition = new EntryPosition(logfilename, logfileoffset, logposTimestamp, serverId);
                        if (logger.isDebugEnabled()) {
                            logger.debug("set {} to be pending start position before finding another proper one...",
                                entryPosition);
                        }
                        logPosition.setPostion(entryPosition);
                        entryPosition.setGtid(entry.getHeader().getGtid());
                    } else if (CanalEntry.EntryType.TRANSACTIONBEGIN.equals(entry.getEntryType())) {
                        // 当前事务开始位点
                        entryPosition = new EntryPosition(logfilename, logfileoffset, logposTimestamp, serverId);
                        if (logger.isDebugEnabled()) {
                            logger.debug("set {} to be pending start position before finding another proper one...",
                                entryPosition);
                        }
                        entryPosition.setGtid(entry.getHeader().getGtid());
                        logPosition.setPostion(entryPosition);
                    }

                    lastPosition = buildLastPosition(entry);
                } catch (Throwable e) {
                    processSinkError(e, lastPosition, searchBinlogFile, 4L);
                }

                return running;
            }
        });
    // ...

}
```
### (9). AbstractEventParser.parseAndProfilingIfNecessary
```
protected CanalEntry.Entry parseAndProfilingIfNecessary(EVENT bod, boolean isSeek) throws Exception {
    long startTs = -1;
    
    boolean enabled = getProfilingEnabled();
    if (enabled) {// false
        startTs = System.currentTimeMillis();
    }

    // ****************************************************************************
    // 10. LogEventConvert.parse
    // ****************************************************************************
    CanalEntry.Entry event = binlogParser.parse(bod, isSeek);
    if (enabled) {
        this.parsingInterval = System.currentTimeMillis() - startTs;
    }

    if (parsedEventCount.incrementAndGet() < 0) {
        parsedEventCount.set(0);
    }
    return event;
}// end parseAndProfilingIfNecessary
```
### (10). LogEventConvert.parse
```
public Entry parse(LogEvent logEvent, boolean isSeek) throws CanalParseException {
    // logEvent = com.taobao.tddl.dbsync.binlog.event.RotateLogEvent
    if (logEvent == null || logEvent instanceof UnknownLogEvent) { // false
        return null;
    }

    // eventType = 4
    int eventType = logEvent.getHeader().getType();

    switch (eventType) {
        case LogEvent.QUERY_EVENT:
            return parseQueryEvent((QueryLogEvent) logEvent, isSeek);
        case LogEvent.XID_EVENT:
            return parseXidEvent((XidLogEvent) logEvent);
        case LogEvent.TABLE_MAP_EVENT:
            break;
        case LogEvent.WRITE_ROWS_EVENT_V1:
        case LogEvent.WRITE_ROWS_EVENT:
            return parseRowsEvent((WriteRowsLogEvent) logEvent);
        case LogEvent.UPDATE_ROWS_EVENT_V1:
        case LogEvent.PARTIAL_UPDATE_ROWS_EVENT:
        case LogEvent.UPDATE_ROWS_EVENT:
            return parseRowsEvent((UpdateRowsLogEvent) logEvent);
        case LogEvent.DELETE_ROWS_EVENT_V1:
        case LogEvent.DELETE_ROWS_EVENT:
            return parseRowsEvent((DeleteRowsLogEvent) logEvent);
        case LogEvent.ROWS_QUERY_LOG_EVENT:
            return parseRowsQueryEvent((RowsQueryLogEvent) logEvent);
        case LogEvent.ANNOTATE_ROWS_EVENT:
            return parseAnnotateRowsEvent((AnnotateRowsEvent) logEvent);
        case LogEvent.USER_VAR_EVENT:
            return parseUserVarLogEvent((UserVarLogEvent) logEvent);
        case LogEvent.INTVAR_EVENT:
            return parseIntrvarLogEvent((IntvarLogEvent) logEvent);
        case LogEvent.RAND_EVENT:
            return parseRandLogEvent((RandLogEvent) logEvent);
        case LogEvent.GTID_LOG_EVENT:
            return parseGTIDLogEvent((GtidLogEvent) logEvent);
        case LogEvent.HEARTBEAT_LOG_EVENT:
            return parseHeartbeatLogEvent((HeartbeatLogEvent) logEvent);
        default:
            break;
    }

    
    // *********************************     
    // RotateLogEvent返回空
    // *********************************     
    return null;
} // end parse
```

### (11). UML图
!["DirectLogFetcher类图"](/assets/canal/imgs/DirectLogFetcher.jpg )

### (11). 总结
>  MysqlConnection.seek执行过程如下:    
> 1. 向MySQL发送dump协议(sendBinlogDump)   
> 2. 创建DirectLogFetcher,并为它设置:SocketChannel.   
> 3. 创建解码器(LogDecoder),并设置支持的事件(LogEvent.ROTATE_EVENT/LogEvent.FORMAT_DESCRIPTION_EVENT/LogEvent.QUERY_EVENT/LogEvent.XID_EVENT)    
> 4. 创建日志上下文(LogContext),它主要用于:在解码过程中设置一些全局的信息.   
> 5. 不断的轮询:DirectLogFetcher.fetch()方法,实则就是读取:SocketChannel里的数据.
> 6. 调用解码器(LogDecoder)进行解码.最终解码为:LogEvent对象的子类.在解码过程中,LogContext会记录一些全局信息(LogPosition/GTIDSet/GtidLogEvent...)   
> 7. 调用Sink函数(SinkFunction)的sink(event)方法,直到该方法返回:false,则跳出第5的轮询.
> 注意:在sink函数里,会调用AbstractEventParser.parseAndProfilingIfNecessary方法,把LogEvent转换成:CanalEntry.Entry.   
> <font color='red'>MySQL协议里的事件产生时间只能精确到秒,倘若发生主从切换,根据时间去定位binlog,还是会有一些问题(增/删语句没什么问题,若是用户并发对同一条数据进行更新,而正好这个时候,出现了主从切换,还是会有问题的).</font>
