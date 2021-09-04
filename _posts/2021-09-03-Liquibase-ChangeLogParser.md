---
layout: post
title: 'Liquibase源码之ChangeLogParser(二)' 
date: 2021-09-03
author: 李新
tags:  Liquibase
---

### (1). 概述
> 前面对Liquibase进行了简单的集成,心里有一个疑问,Liquibase是如何把ChangLog(XML/JSON/YAML/SQLFile)进行解析,并执行的?    
> 原理就在ChangeLogParser

### (2). ChangeLogParser部析
> 从类图上就能看出来,ChangeLogParser支持:XML/JSON/YAML/SQLFile,我只对XML的实现感兴趣.      

!["ChangeLogParser"](/assets/liquibase/imgs/ChangeLogParser-Class.jpg)

### (3). AbstractChangeLogParser
```
public abstract class AbstractChangeLogParser implements ChangeLogParser {

    @Override
    public DatabaseChangeLog parse(String physicalChangeLogLocation, ChangeLogParameters changeLogParameters,
                                   ResourceAccessor resourceAccessor) throws ChangeLogParseException {
        
		// *************************************************************************************
		// 1. 委托给子类:XMLChangeLogSAXParser对xml进行解析.
		// *************************************************************************************
		ParsedNode parsedNode = parseToNode(physicalChangeLogLocation, changeLogParameters, resourceAccessor);
        if (parsedNode == null) {
            return null;
        }


		// *************************************************************************************
		// 2. 委托给:DatabaseChangeLog对xml解析后的内容,进行深度解析成业务模型
		// *************************************************************************************
        DatabaseChangeLog changeLog = new DatabaseChangeLog(physicalChangeLogLocation);
        changeLog.setChangeLogParameters(changeLogParameters);
        try {
			// 
            changeLog.load(parsedNode, resourceAccessor);
        } catch (Exception e) {
            throw new ChangeLogParseException(e);
        }

        return changeLog;
    }

    protected abstract ParsedNode parseToNode(String physicalChangeLogLocation, ChangeLogParameters changeLogParameters,
                                              ResourceAccessor resourceAccessor) throws ChangeLogParseException;
}
```
### (4). XML解析入口:XMLChangeLogSAXParser
```
public class XMLChangeLogSAXParser extends AbstractChangeLogParser {
    
	public static final String LIQUIBASE_SCHEMA_VERSION = "3.6";
    private static final boolean PREFER_INTERNAL_XSD = Boolean.getBoolean("liquibase.prefer.internal.xsd");
    private static final String XSD_FILE = "dbchangelog-" + LIQUIBASE_SCHEMA_VERSION + ".xsd";
    private SAXParserFactory saxParserFactory;

    public XMLChangeLogSAXParser() {
        saxParserFactory = SAXParserFactory.newInstance();
        saxParserFactory.setValidating(true);
        saxParserFactory.setNamespaceAware(true);
        
        if (PREFER_INTERNAL_XSD) {
            InputStream xsdInputStream = XMLChangeLogSAXParser.class.getResourceAsStream(XSD_FILE);
            if (xsdInputStream != null) {
                try {
                    SchemaFactory schemaFactory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
                    Schema schema = schemaFactory.newSchema(new StreamSource(xsdInputStream));
                    saxParserFactory.setSchema(schema);
                    saxParserFactory.setValidating(false);
                } catch (SAXException e) {
                    LogService.getLog(XMLChangeLogSAXParser.class).warning("Could not load " + XSD_FILE + ", enabling parser validator", e);
                }
            }
        }
    }

    
    //  *********************************************************************************************************
	// 1. 对xml文件进行解析.
	//  *********************************************************************************************************
	@Override
    protected ParsedNode parseToNode(String physicalChangeLogLocation, ChangeLogParameters changeLogParameters,ResourceAccessor resourceAccessor) throws ChangeLogParseException {
        try (
			// 加载xml文佣
            InputStream inputStream = StreamUtil.singleInputStream(physicalChangeLogLocation, resourceAccessor)) {
            SAXParser parser = saxParserFactory.newSAXParser();
            trySetSchemaLanguageProperty(parser);
    
            XMLReader xmlReader = parser.getXMLReader();
            LiquibaseEntityResolver resolver=new LiquibaseEntityResolver(this);
            resolver.useResoureAccessor(resourceAccessor,FilenameUtils.getFullPath(physicalChangeLogLocation));
            xmlReader.setEntityResolver(resolver);
            xmlReader.setErrorHandler(new ErrorHandler() {
                @Override
                public void warning(SAXParseException exception) throws SAXException {
                    LogService.getLog(getClass()).warning(LogType.LOG, exception.getMessage());
                    throw exception;
                }

                @Override
                public void error(SAXParseException exception) throws SAXException {
                    LogService.getLog(getClass()).severe(LogType.LOG, exception.getMessage());
                    throw exception;
                }

                @Override
                public void fatalError(SAXParseException exception) throws SAXException {
                    LogService.getLog(getClass()).severe(LogType.LOG, exception.getMessage());
                    throw exception;
                }
            });
        	
            if (inputStream == null) {
                if (physicalChangeLogLocation.startsWith("WEB-INF/classes/")) {
                    // Correct physicalChangeLogLocation and try again.
                    return parseToNode(
                        physicalChangeLogLocation.replaceFirst("WEB-INF/classes/", ""),
                            changeLogParameters, resourceAccessor);
                } else {
                    throw new ChangeLogParseException(physicalChangeLogLocation + " does not exist");
                }
            }

            XMLChangeLogSAXHandler contentHandler = new XMLChangeLogSAXHandler(physicalChangeLogLocation, resourceAccessor, changeLogParameters);
            xmlReader.setContentHandler(contentHandler);
            xmlReader.parse(new InputSource(new UtfBomStripperInputStream(inputStream)));

            return contentHandler.getDatabaseChangeLogTree();
        } catch (ChangeLogParseException e) {
            throw e;
        } catch (IOException e) {
            throw new ChangeLogParseException("Error Reading Migration File: " + e.getMessage(), e);
        } catch (SAXParseException e) {
            throw new ChangeLogParseException("Error parsing line " + e.getLineNumber() + " column " + e.getColumnNumber() + " of " + physicalChangeLogLocation +": " + e.getMessage(), e);
        } catch (SAXException e) {
            Throwable parentCause = e.getException();
            while (parentCause != null) {
                if (parentCause instanceof ChangeLogParseException) {
                    throw ((ChangeLogParseException) parentCause);
                }
                parentCause = parentCause.getCause();
            }
            String reason = e.getMessage();
            String causeReason = null;
            if (e.getCause() != null) {
                causeReason = e.getCause().getMessage();
            }
            if (reason == null) {
                if (causeReason != null) {
                    reason = causeReason;
                } else {
                    reason = "Unknown Reason";
                }
            }

            throw new ChangeLogParseException("Invalid Migration File: " + reason, e);
        } catch (Exception e) {
            throw new ChangeLogParseException(e);
        }
    } // end parseToNode

}
```
### (5). 解析XML转换成DatabaseChangeLog
```
public class DatabaseChangeLog implements Comparable<DatabaseChangeLog>, Conditional {

   // *****************************************************************************
   // 1. 解析xml内容,并转换到DatabaseChangeLog里
   // *****************************************************************************
	public void load(ParsedNode parsedNode, ResourceAccessor resourceAccessor)
            throws ParsedNodeException, SetupException {
        
		// 1.1 获得日志文件的详细路径
		setLogicalFilePath(parsedNode.getChildValue(null, "logicalFilePath", String.class));

		// 1.2 解析context
        setContexts(new ContextExpression(parsedNode.getChildValue(null, "context", String.class)));
        String objectQuotingStrategy = parsedNode.getChildValue(null, "objectQuotingStrategy", String.class);
        if (objectQuotingStrategy != null) {
            setObjectQuotingStrategy(ObjectQuotingStrategy.valueOf(objectQuotingStrategy));
        }

		// 1.3 遍历所有的xml子节点
        for (ParsedNode childNode : parsedNode.getChildren()) {
            handleChildNode(childNode, resourceAccessor);
        }
    } //end load

    
   
	protected void handleChildNode(ParsedNode node, ResourceAccessor resourceAccessor)
            throws ParsedNodeException, SetupException {
        expandExpressions(node);

        String nodeName = node.getName();
        switch (nodeName) {  
            //  *************************************************************************************************
			// 1.4 对<changeSet>进行解析
			//  *************************************************************************************************
			case"changeSet":  
                // 如果dbms match
				if (isDbmsMatch(node.getChildValue(null, "dbms", String.class))) {
					// 
					this.addChangeSet(createChangeSet(node, resourceAccessor));
				}
                break;
            //  *************************************************************************************************
			// 1.5 对<include>进行解析
			//  *************************************************************************************************
			case"include": {
            String path = node.getChildValue(null, "file", String.class);
            if (path == null) {
                throw new UnexpectedLiquibaseException("No 'file' attribute on 'include'");
            }
            path = path.replace('\\', '/');
            ContextExpression includeContexts = new ContextExpression(node.getChildValue(null, "context", String.class));
            try {
                include(path, node.getChildValue(null, "relativeToChangelogFile", false), resourceAccessor, includeContexts, true);
            } catch (LiquibaseException e) {
                throw new SetupException(e);}
                break;
            }
			//  *************************************************************************************************
			// 1.6 对<includeAll>进行解析
			//  *************************************************************************************************
            case "includeAll": {
                String path = node.getChildValue(null, "path", String.class);
                String resourceFilterDef = node.getChildValue(null, "filter", String.class);
                if (resourceFilterDef == null) {
                    resourceFilterDef = node.getChildValue(null, "resourceFilter", String.class);
                }
                IncludeAllFilter resourceFilter = null;
                if (resourceFilterDef != null) {
                    try {
                        resourceFilter = (IncludeAllFilter) Class.forName(resourceFilterDef).getConstructor().newInstance();
                    } catch (ReflectiveOperationException e) {
                        throw new SetupException(e);
                    }
                }

                String resourceComparatorDef = node.getChildValue(null, "resourceComparator", String.class);
                Comparator<String> resourceComparator = null;
                if (resourceComparatorDef != null) {
                    try {
                        resourceComparator = (Comparator<String>) Class.forName(resourceComparatorDef).getConstructor().newInstance();
                    } catch (ReflectiveOperationException e) {
                        //take default comparator
                        LogService.getLog(getClass()).info(LogType.LOG, "no resourceComparator defined - taking default " +
                         "implementation");
                        resourceComparator = getStandardChangeLogComparator();
                    }
                }

                ContextExpression includeContexts = new ContextExpression(node.getChildValue(null, "context", String.class));
                includeAll(path, node.getChildValue(null, "relativeToChangelogFile", false), resourceFilter,
                        node.getChildValue(null, "errorIfMissingOrEmpty", true),
                        resourceComparator, resourceAccessor, includeContexts);
                break;
            }
			//  *************************************************************************************************
			// 1.7 对<preConditions>进行解析
			//  *************************************************************************************************
            case "preConditions": {
                this.preconditionContainer = new PreconditionContainer();
                try {
                    this.preconditionContainer.load(node, resourceAccessor);
                } catch (ParsedNodeException e) {
                    e.printStackTrace();
                }
                break;
            }

			//  *************************************************************************************************
			// 1.8 对<property>进行解析
			//  *************************************************************************************************
            case "property": {
                try {
                    String context = node.getChildValue(null, "context", String.class);
                    String dbms = node.getChildValue(null, "dbms", String.class);
                    String labels = node.getChildValue(null, "labels", String.class);
                    Boolean global = node.getChildValue(null, "global", Boolean.class);
                    if (global == null) {
                        // okay behave like liquibase < 3.4 and set global == true
                        global = true;
                    }

                    String file = node.getChildValue(null, "file", String.class);

                    if (file == null) {
                        // direct referenced property, no file
                        String name = node.getChildValue(null, "name", String.class);
                        String value = node.getChildValue(null, "value", String.class);

                        this.changeLogParameters.set(name, value, context, labels, dbms, global, this);
                    } else {
                        // read properties from the file
                        Properties props = new Properties();
                        InputStream propertiesStream = StreamUtil.singleInputStream(file, resourceAccessor);
                        if (propertiesStream == null) {
                            LogService.getLog(getClass()).info(LogType.LOG, "Could not open properties file " + file);
                        } else {
                            props.load(propertiesStream);

                            for (Map.Entry entry : props.entrySet()) {
                                this.changeLogParameters.set(
                                        entry.getKey().toString(),
                                        entry.getValue().toString(),
                                        context,
                                        labels,
                                        dbms,
                                        global,
                                        this
                                );
                            }
                        }
                    }
                } catch (IOException e) {
                    throw new ParsedNodeException(e);
                }

                break;
            }
        }
    } // end handleChildNode


	// ****************************************************************
	// 1.4.1 对<changeSet>进行解析,并转换成:ChangeSet对象
	// ****************************************************************
	protected ChangeSet createChangeSet(ParsedNode node, ResourceAccessor resourceAccessor) throws ParsedNodeException {
		ChangeSet changeSet = new ChangeSet(this);
		changeSet.setChangeLogParameters(this.getChangeLogParameters());
		changeSet.load(node, resourceAccessor);
		return changeSet;
	}
}	
```
### (6). ChangeSet对象
```
public class ChangeSet implements Conditional, ChangeLogChild {

    // ********************************************************************************
	// 1. 创建ChangeSet,解析xml,并把xml内容转换到:ChangeSet对象里.
	// ********************************************************************************
    public void load(ParsedNode node, ResourceAccessor resourceAccessor) throws ParsedNodeException {
		// <changeSet id="1">
        this.id = node.getChildValue(null, "id", String.class);
		// <changeSet author="nvoxland">
        this.author = node.getChildValue(null, "author", String.class);
		// runAlways
        this.alwaysRun  = node.getChildValue(null, "runAlways", node.getChildValue(null, "alwaysRun", false));
		// runOnChange
        this.runOnChange  = node.getChildValue(null, "runOnChange", false);
		// context
        this.contexts = new ContextExpression(node.getChildValue(null, "context", String.class));
		// 
        this.labels = new Labels(StringUtils.trimToNull(node.getChildValue(null, "labels", String.class)));

		// <changeSet id="1" author="nvoxland" dbms="" >
        setDbms(node.getChildValue(null, "dbms", String.class));
		// 
        this.runInTransaction  = node.getChildValue(null, "runInTransaction", true);
        this.created = node.getChildValue(null, "created", String.class);
        this.runOrder = node.getChildValue(null, "runOrder", String.class);
        this.ignore = node.getChildValue(null, "ignore", false);
        this.comments = StringUtils.join(node.getChildren(null, "comment"), "\n", new StringUtils.StringUtilsFormatter() {
            @Override
            public String toString(Object obj) {
                if (((ParsedNode) obj).getValue() == null) {
                    return "";
                } else {
                    return ((ParsedNode) obj).getValue().toString();
                }
            }
        });
        this.comments = StringUtils.trimToNull(this.comments);

        String objectQuotingStrategyString = StringUtils.trimToNull(node.getChildValue(null, "objectQuotingStrategy", String.class));
        if (changeLog != null) {
            this.objectQuotingStrategy = changeLog.getObjectQuotingStrategy();
        }
        if (objectQuotingStrategyString != null) {
            this.objectQuotingStrategy = ObjectQuotingStrategy.valueOf(objectQuotingStrategyString);
        }

        if (this.objectQuotingStrategy == null) {
            this.objectQuotingStrategy = ObjectQuotingStrategy.LEGACY;
        }

        // xml文件位置
        this.filePath = StringUtils.trimToNull(node.getChildValue(null, "logicalFilePath", String.class));
        if (filePath == null) {
            filePath = changeLog.getFilePath();
        }

        this.setFailOnError(node.getChildValue(null, "failOnError", Boolean.class));
        String onValidationFailString = node.getChildValue(null, "onValidationFail", "HALT");
        this.setOnValidationFail(ValidationFailOption.valueOf(onValidationFailString));

        for (ParsedNode child : node.getChildren()) {  // 循环解析余下的所有子节点
		    // ********************************************************************************
			// 2. 解析<changeSet> ...  </changeSet>下所有的内容.
			// ********************************************************************************
            handleChildNode(child, resourceAccessor);
        }
    } // end load

    // ********************************************************************************
	// 3. 解析<changeSet>下所有内容.
	// ********************************************************************************
	protected void handleChildNode(ParsedNode child, ResourceAccessor resourceAccessor) throws ParsedNodeException {
        switch (child.getName()) {
            case "rollback":    // 解析rollback
                handleRollbackNode(child, resourceAccessor);
                break;
            case "validCheckSum":
            case "validCheckSums":   // 解析validCheckSums
                if (child.getValue() == null) {
                    return;
                }

                if (child.getValue() instanceof Collection) {
                    for (Object checksum : (Collection) child.getValue()) {
                        addValidCheckSum((String) checksum);
                    }
                } else {
                    addValidCheckSum(child.getValue(String.class));
                }
                break;
            case "modifySql":   // 解析modifySql
                String dbmsString = StringUtils.trimToNull(child.getChildValue(null, "dbms", String.class));
                String contextString = StringUtils.trimToNull(child.getChildValue(null, "context", String.class));
                String labelsString = StringUtils.trimToNull(child.getChildValue(null, "labels", String.class));
                boolean applyToRollback = child.getChildValue(null, "applyToRollback", false);

                Set<String> dbms = new HashSet<>();
                if (dbmsString != null) {
                    dbms.addAll(StringUtils.splitAndTrim(dbmsString, ","));
                }
                ContextExpression context = null;
                if (contextString != null) {
                    context = new ContextExpression(contextString);
                }

                Labels labels = null;
                if (labelsString != null) {
                    labels = new Labels(labelsString);
                }


                List<ParsedNode> potentialVisitors = child.getChildren();
                for (ParsedNode node : potentialVisitors) {
                    SqlVisitor sqlVisitor = SqlVisitorFactory.getInstance().create(node.getName());
                    if (sqlVisitor != null) {
                        sqlVisitor.setApplyToRollback(applyToRollback);
                        if (!dbms.isEmpty()) {
                            sqlVisitor.setApplicableDbms(dbms);
                        }
                        sqlVisitor.setContexts(context);
                        sqlVisitor.setLabels(labels);
                        sqlVisitor.load(node, resourceAccessor);

                        addSqlVisitor(sqlVisitor);
                    }
                }


                break;
            case "preConditions":  // 解析preConditions
                this.preconditions = new PreconditionContainer();
                try {
                    this.preconditions.load(child, resourceAccessor);
                } catch (ParsedNodeException e) {
                    e.printStackTrace();
                }
                break;
            case "changes":   // 解析changes
                for (ParsedNode changeNode : child.getChildren()) {
                    handleChildNode(changeNode, resourceAccessor);
                }
                break;
            default:        
			    // *********************************************************************************
				// 4. 解析其它(createTable/addColumn/insert/...)标签.
				// *********************************************************************************
                Change change = toChange(child, resourceAccessor);
                if ((change == null) && (child.getValue() instanceof String)) {
                    this.setAttribute(child.getName(), child.getValue());
                } else {
                    addChange(change);
                }
                break;
        }
    }// end handleChildNode

    
	// ************************************************************************
	// 5. 解析其它(createTable/addColumn/insert/...)标签,通过Change进行承载
	// ************************************************************************
	protected Change toChange(ParsedNode value, ResourceAccessor resourceAccessor) throws ParsedNodeException {
		// **********************************************************************
		// 通过SPI,加载(liquibase.change.Change)的所有实现类,这样就能实现动态解析XML的标签了
        // **********************************************************************
		Change change = ChangeFactory.getInstance().create(value.getName());
        if (change == null) {
            return null;
        } else {
			// **********************************************************************
			// 调用Change.load方法,解析所有的标签,在这里以:InsertDataChange为案例
			// **********************************************************************
            change.load(value, resourceAccessor);

            return change;
        }
    } // end toChange

}
```
### (7). InsertDataChange

```
<!-- 对insert标签进行解析 -->
<changeSet id="5" author="nvoxland" context="test">
        <insert tableName="person">
            <column name="firstname" value="John"/>
            <column name="lastname" value="Doe"/>
            <column name="username" value="jdoe"/>
        </insert>
</changeSet>
```


```
package liquibase.change.core;

@DatabaseChange(name="insert", description = "Inserts data into an existing table", priority = ChangeMetaData.PRIORITY_DEFAULT, appliesTo = "table")
public class InsertDataChange extends AbstractChange implements ChangeWithColumns<ColumnConfig>, DbmsTargetedChange {

    private String catalogName;
    private String schemaName;
    private String tableName;
    private List<ColumnConfig> columns;
    private String dbms;

    // ... 
}
```
### (8). 总结
> 通过对源码的剖析,我们能知道Liquibase是允许我们自定义标签,并解析的,后面会详细剖析自动定义标签以及Liquibase是如何把Change进行转换并执行的.  