---
layout: post
title: 'Calcite CSV源码深入FilterableTable(四)'
date: 2021-04-11
author: 李新
tags:  Calcite
---

### (1). 概述
> 在这一小节,部析Calcite是如何通过SQL检索CSV的过程.

### (2). CsvTest

```
package help.lixin.calcite;

import java.io.File;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;
import java.util.Properties;

import org.apache.calcite.adapter.csv.CsvSchema;
import org.apache.calcite.adapter.csv.CsvTable;
import org.apache.calcite.jdbc.CalciteConnection;
import org.apache.calcite.schema.SchemaPlus;
import org.junit.Test;

public class CsvTest {

	@Test
	public void testQuery() throws Exception {
		// 0.获取csv文件的路径，注意获取到文件所在上层路径就可以了
		String path = CsvTest.class.getClassLoader().getResource("sales").getPath();

		// 1.构建CsvSchema对象，在Calcite中，不同数据源对应不同Schema，比如CsvSchema、DruidSchema、ElasticsearchSchema等
		CsvSchema csvSchema = new CsvSchema(new File(path), CsvTable.Flavor.FILTERABLE);

		// 2.构建Connection
		// 2.1 设置连接参数
		Properties info = new Properties();
		// 不区分sql大小写
		info.setProperty("caseSensitive", "false");
		// 2.2 获取标准的JDBC Connection
		Connection connection = DriverManager.getConnection("jdbc:calcite:", info);
		// 2.3 获取Calcite封装的Connection
		CalciteConnection calciteConnection = connection.unwrap(CalciteConnection.class);

		// 3.构建RootSchema，在Calcite中，RootSchema是所有数据源schema的parent，多个不同数据源schema可以挂在同一个RootSchema下
		// 以实现查询不同数据源的目的
		SchemaPlus rootSchema = calciteConnection.getRootSchema();

		// 4.将不同数据源schema挂载到RootSchema，这里添加CsvSchema
		rootSchema.add("csv", csvSchema);

		// 5.执行SQL查询，通过SQL方式访问csv文件
		String sql = "select * from csv.depts where name = 'Marketing' OR deptno = 30";
		Statement statement = calciteConnection.createStatement();
		ResultSet resultSet = statement.executeQuery(sql);

		// 6.遍历打印查询结果集
		System.out.println(ResultSetUtil.resultString(resultSet));
	}
}
```
### (3). sales/DEPTS.csv
```
DEPTNO:int,NAME:string
10,"Sales"
20,"Marketing"
30,"Accounts"
```
### (4). 源码入口在哪?
> 我们知道Calcite是通过SQL进行检索CSV文件的,同时,它抽象出来了两个概念:Schema/Table,所以,我们要找到Schema入口.就是上面代码类:CsvSchema

```
// 1. Schema
public interface Schema {
	// ....
}

// 2. AbstractSchema对Schema进行了实现
public class AbstractSchema implements Schema {
   // ...
   // 获得所有的Table信息.
   protected Map<String, Table> getTableMap() {
       return ImmutableMap.of();
   }
}

// 3. CsvSchema继承了:AbstractSchema,并且,重写:getTableMap
public class CsvSchema extends AbstractSchema {
  // 4. csv所在的目录,无须定位具体某个csv文件
  private final File directoryFile;
  // 5. SCANNABLE, FILTERABLE, TRANSLATABLE
  // 全表 / 过滤  / RelNode
  private final CsvTable.Flavor flavor;
  // 6. 所有表的元数据信息
  private Map<String, Table> tableMap;
  
  // ... ...
  
  protected Map<String, Table> getTableMap() {
      if (tableMap == null) {
		  // 7. 扫描目录,并全创建
        tableMap = createTableMap();
      }
      return tableMap;
  }// end getTableMap
  
  private Map<String, Table> createTableMap() {
	  final Source baseSource = Sources.of(directoryFile);
	  // 8. 过滤目录下:.gz/.csv/.json的文件
	  File[] files = directoryFile.listFiles((dir, name) -> {
		final String nameSansGz = trim(name, ".gz");
		return nameSansGz.endsWith(".csv")
			|| nameSansGz.endsWith(".json");
	  });
	  
	  // 9. 目录下不存在这些文件,则什么都不做
	  if (files == null) {
		System.out.println("directory " + directoryFile + " not found");
		files = new File[0];
	  }
	  
	  // Build a map from table name to table; each file becomes a table.
	  final ImmutableMap.Builder<String, Table> builder = ImmutableMap.builder();
	  // 10. 遍历所有的文件
	  for (File file : files) {
		Source source = Sources.of(file);
		// 11. 针对*.gz处理
		Source sourceSansGz = source.trim(".gz");
		// 12. 针对.json处理
		final Source sourceSansJson = sourceSansGz.trimOrNull(".json");
		if (sourceSansJson != null) {  // 针对*.json处理
		  final Table table = new JsonScannableTable(source);
		  builder.put(sourceSansJson.relative(baseSource).path(), table);
		}
		final Source sourceSansCsv = sourceSansGz.trimOrNull(".csv");
		if (sourceSansCsv != null) { // 针对*.csv处理
		  // ******************************************************************
		  // 13. 根据*.csv创建Table
		  // ******************************************************************
		  final Table table = createTable(source);
		  builder.put(sourceSansCsv.relative(baseSource).path(), table);
		}
	  }
	  return builder.build();
  }// end createTableMap
  
  
  private Table createTable(Source source) {
	  switch (flavor) {
	  case TRANSLATABLE:  
		return new CsvTranslatableTable(source, null);
	  case SCANNABLE:
		return new CsvScannableTable(source, null);
	  case FILTERABLE: 
		 // ********************************************************************
		 // 带有谓词下推的处理.
		 // ********************************************************************
		return new CsvFilterableTable(source, null);
	  default:
		throw new AssertionError("Unknown flavor " + this.flavor);
	  }
	} // end createTable
 
}
```
### (5). CsvFilterableTable
```
public class CsvFilterableTable 
      // 1. 这是具体的某一张"表"了
      extends CsvTable
	  // 2. 带有条件的表过滤
      implements FilterableTable {

   public Enumerable<Object[]> scan(DataContext root, List<RexNode> filters) {
	   // 3. 读取csv文件的第一行,获得每一列的信息(列名称,数据类型)
      final List<CsvFieldType> fieldTypes = getFieldTypes(root.getTypeFactory());
      final String[] filterValues = new String[fieldTypes.size()];
	  // *****************************************************************************
	  // 4. 如果查询是带有条件的(select * from csv.depts where name = 'Marketing')
	  // filterValues = [null,"Marketing"],实际是代表要过滤的条件
	  // 注意:filterValues是数组形式,name在csv中的第一列,所以,过滤条件内容是在数组的第一列,而第0列为空.
	  // 通过谓词下推,让数据源在读取数据时,就减少数据的返回.这样在"投影"时,就能更多的节约内存.
	  // 而不是所有数据全部加载到内存,再通过内存去减少数据.
	  // *****************************************************************************
      filters.removeIf(filter -> addFilter(filter, filterValues));
	  
      final List<Integer> fields = ImmutableIntList.identity(fieldTypes.size());
      final AtomicBoolean cancelFlag = DataContext.Variable.CANCEL_FLAG.get(root);
	  
	  // *********************************************************************
	  // 5. 通过CsvEnumerator实现csv的读取,注意,返回的是:Enumerator<Object[]>
	  // *********************************************************************
      return new AbstractEnumerable<Object[]>() {
        public Enumerator<Object[]> enumerator() {
          return new CsvEnumerator<>(source, cancelFlag, false, filterValues,
              CsvEnumerator.arrayConverter(fieldTypes, fields, false));
        }
      };
   }

   private boolean addFilter(RexNode filter, Object[] filterValues) {
     if (filter.isA(SqlKind.AND)) {
        // We cannot refine(remove) the operands of AND,
        // it will cause o.a.c.i.TableScanNode.createFilterable filters check failed.
      ((RexCall) filter).getOperands().forEach(subFilter -> addFilter(subFilter, filterValues));
     } else if (filter.isA(SqlKind.EQUALS)) {
       final RexCall call = (RexCall) filter;
       RexNode left = call.getOperands().get(0);
       if (left.isA(SqlKind.CAST)) {
         left = ((RexCall) left).operands.get(0);
       }
	   
       final RexNode right = call.getOperands().get(1);
       if (left instanceof RexInputRef
          && right instanceof RexLiteral) {
        final int index = ((RexInputRef) left).getIndex();
        if (filterValues[index] == null) {
          filterValues[index] = ((RexLiteral) right).getValue2().toString();
          return true;
        } // end if
      } // end if
    } // end else if
    return false;
  } // end addFilter

}
```
### (6). CsvEnumerator
```
public class CsvEnumerator<E> implements Enumerator<E> {
	public CsvEnumerator(Source source, AtomicBoolean cancelFlag, boolean stream,
	      String[] filterValues, RowConverter<E> rowConverter) {
	    this.cancelFlag = cancelFlag;
		// 1. 数据行转换器
	    this.rowConverter = rowConverter;
		// 2. 过滤条件
	    this.filterValues = filterValues == null ? null
	        : ImmutableNullableList.copyOf(filterValues);
	    try {
		  // 是否流式处理
	      if (stream) {  // false
	        this.reader = new CsvStreamReader(source);
	      } else { // 3. 打开csv文件
	        this.reader = openCsv(source);
	      }
		  // 4. 跳过第一行
	      this.reader.readNext(); // skip header row
	    } catch (IOException e) {
	      throw new RuntimeException(e);
	    }
	} // end 
	
	
	// *******************************************************************
	// 5. while(resultSet.next()){ // ...  }
	//    当我们通过ResultSet去检索数据时,实则,是在读取csv文件,而且还是一行一行的读.
	// *******************************************************************
	public boolean moveNext() {
	    try {
	    outer:
	      for (;;) {
	        if (cancelFlag.get()) {
	          return false;
	        }
			// 5.1 通过csv读取一行数据
	        final String[] strings = reader.readNext();
			// 5.2 读取的数据不存在的情况下
	        if (strings == null) {
				// 5.3 判断是否为流式读取
	          if (reader instanceof CsvStreamReader) {
	            try {
					// 5.4 休眠2秒针
	              Thread.sleep(CsvStreamReader.DEFAULT_MONITOR_DELAY);
	            } catch (InterruptedException e) {
	              throw new RuntimeException(e);
	            }
				// 5.5 跳到:for循环重新开始读取
	            continue;
	          }
			  // 5.6 如果不是流式读取,则代表读取到EOF了
	          current = null;
			  // 关闭流
	          reader.close();
			  // 返回false
	          return false;
	        } // end 读取EOF或Stream的情况下
			
			
	        if (filterValues != null) {  // 5.7 需要在读取时就过滤的数据
			  //  遍历读取的CSV一行数据(10,"Sales")
	          for (int i = 0; i < strings.length; i++) {  
				// 判断是否为要过滤的列
				// *********************************************************************
				// 这也是 WHERE deptno = 1 OR name = 'xxx'不走索引的原理
				// *********************************************************************
				// filterValues=[null,"Marketing"]
				//              DEPTNO:int,NAME:string
	            String filterValue = filterValues.get(i);
	            if (filterValue != null) { 
				   // 如果读取的数据列(strings[i])与filterValue不相等,则跳过数据解析阶段,继续for循环.
	              if (!filterValue.equals(strings[i])) {
	                continue outer;
	              }
	            }
	          }
	        } // 根据谓词下推,在读取数据时,就对数据进行过滤
			
			// 5.8 通过RowConverter对读取的数据行进行解析.
	        current = rowConverter.convertRow(strings);
	        return true;
	      }
	    } catch (IOException e) {
	      throw new RuntimeException(e);
	    }
	} // end moveNext
	
}
```
### (7). 总结
> 总体来说,对CSV的解析有了一个大体的入门,也可以参数上面的方式,自定义Schema和Table.    
> <font color='red'>谓词下推确实是能有效解决数据的返回,但是,对IO的操作呢?实际仍然是一行一行的读取,所以,这也是为什么数据库里要有索引的存在的原因了.</font>         
> <font color='red'>对Calcite的使用:还是要尽可能的Hold住场景,它比较适合那些:索引在ES,数据在HBase.因为,这些数据源充当了索引.又或者说它适合于OLAP的场景,非实时的场景.</font>        