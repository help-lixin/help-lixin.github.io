---
layout: post
title: 'Lucene 高级查询(七)'
date: 2018-09-10
author: 李新
tags: Lucene
---

### (1). 根据文本内容进行检索(QueryParser)
> QueryParser只能对文本进行检索,是不能对数值范围这样的Field进行检索的.   
> SELECT * FROM xxx WHERE pname LIKE '%真皮%' OR pname LIKE '%纹圆%';   

```
@Test
public void tetsQuery() throws Exception {
	Analyzer analyzer = new IKAnalyzer();

	// 2.创建查询对象
	QueryParser queryParser = new QueryParser("pname", analyzer);

	// 3.设置搜索关键词
	// SELECT * FROM xxx WHERE pname LIKE '%真皮%' OR pname LIKE '%纹圆%';
	// pname:真皮
	// 默认情况下:词汇之间是OR,求的是交集.
	Query query = queryParser.parse("真皮 OR 纹圆");

	// 4. 指定索引库位置
	String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
	Directory directory = FSDirectory.open(Paths.get(indexDir));

	// 5. 打开索引库,并转换成:IndexReader
	IndexReader reader = DirectoryReader.open(directory);

	// 6. 创建搜索对象
	IndexSearcher searcher = new IndexSearcher(reader);

	// 7. 搜索并返回结果
	TopDocs topDocs = searcher.search(query, 1000);

	// 8. 转换结果集
	long totalHits = topDocs.totalHits;
	ScoreDoc[] docs = topDocs.scoreDocs;
	Optional<ScoreDoc[]> optional =  Optional.of(docs);
	optional.ifPresent((item)->{
		for(ScoreDoc doc : item) {
			// Lucene为Document分配的唯一ID,并非业务:pid
			int docID = doc.doc;
			try {
				Document document = reader.document(docID);
				System.out.println("==================================================");
				System.out.println("  pid: "+document.get("pid"));
				System.out.println("  pname: "+document.get("pname"));
			} catch (IOException e) {
			}
		}
	});
	
	System.err.println("totalHits:" + totalHits);
	// 9. 释放资源
	reader.close();
}
```
### (2). 根据数值范围进行检索(***Point)
> DoublePoint/LongPoint/...
> SELECT * FROM xxx WHERE price >= 100 AND price <= 200;

```
@Test
public void testBetweenQuery() throws Exception {
	// 包含起始值和结束值的
	// SELECT * FROM xxx WHERE price >= 100 AND price <= 200;
	// 2.创建查询对象
	Query query  = DoublePoint.newRangeQuery("price", 100, 200);

	// 4. 指定索引库位置
	String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
	Directory directory = FSDirectory.open(Paths.get(indexDir));

	// 5. 打开索引库,并转换成:IndexReader
	IndexReader reader = DirectoryReader.open(directory);

	// 6. 创建搜索对象
	IndexSearcher searcher = new IndexSearcher(reader);

	// 7. 搜索并返回结果
	TopDocs topDocs = searcher.search(query, 1000);

	// 8. 转换结果集
	long totalHits = topDocs.totalHits;
	ScoreDoc[] docs = topDocs.scoreDocs;
	Optional<ScoreDoc[]> optional =  Optional.of(docs);
	optional.ifPresent((item)->{
		for(ScoreDoc doc : item) {
			// Lucene为Document分配的唯一ID,并非业务:pid
			int docID = doc.doc;
			try {
				Document document = reader.document(docID);
				System.out.println("==================================================");
				System.out.println("  pid: "+document.get("pid"));
				System.out.println("  pname: "+document.get("pname"));
				System.out.println("  price: "+document.get("price"));
			} catch (IOException e) {
			}
		}
	});
	
	System.err.println("totalHits:" + totalHits);
	// 9. 释放资源
	reader.close();
}


/**
* 根据主键查询
* @throws Exception
*/
@Test
public void testPrimaryKeyQuery() throws Exception {
	// 包含起始值和结束值的.
	// SELECT * FROM xxx WHERE price >= 100 AND price <= 200;
	// 2.创建查询对象
	Query query  = LongPoint.newExactQuery("pid", 316);

	// 4. 指定索引库位置
	String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
	Directory directory = FSDirectory.open(Paths.get(indexDir));

	// 5. 打开索引库,并转换成:IndexReader
	IndexReader reader = DirectoryReader.open(directory);

	// 6. 创建搜索对象
	IndexSearcher searcher = new IndexSearcher(reader);

	// 7. 搜索并返回结果
	TopDocs topDocs = searcher.search(query, 1000);

	// 8. 转换结果集
	long totalHits = topDocs.totalHits;
	ScoreDoc[] docs = topDocs.scoreDocs;
	Optional<ScoreDoc[]> optional =  Optional.of(docs);
	optional.ifPresent((item)->{
		for(ScoreDoc doc : item) {
			// Lucene为Document分配的唯一ID,并非业务:pid
			int docID = doc.doc;
			try {
				Document document = reader.document(docID);
				System.out.println("==================================================");
				System.out.println("  pid: "+document.get("pid"));
				System.out.println("  pname: "+document.get("pname"));
				System.out.println("  price: "+document.get("price"));
			} catch (IOException e) {
			}
		}
	});
	
	System.err.println("totalHits:" + totalHits);
	// 9. 释放资源
	reader.close();
}
```
### (3). 多条件组合检索
> SELECT * FROM xxx WHERE pname LIKE '%真皮%' AND (price >= 10 AND price<=60 );

```
@Test
public void testComposeQuery() throws Exception{
	Analyzer analyzer = new IKAnalyzer();

	// 2.创建查询对象
	QueryParser queryParser = new QueryParser("pname", analyzer);

	// 3.设置搜索关键词
	// SELECT * FROM xxx WHERE pname LIKE '%真皮%' AND (price >= 10 AND price<=60 );
	// pname:真皮
	// 默认情况下:词汇之间是OR,求的是交集.
	Query likeQuery = queryParser.parse("真皮");
	// price >= 100 and price <= 200
	Query betweenQuery = DoublePoint.newRangeQuery("price", 10, 60);

	BooleanQuery query = new BooleanQuery.Builder() //
			// MUST == AND
			// SHOULD == OR
			// MUST_NOT == NOT  
			.add(likeQuery, Occur.MUST)
			.add(betweenQuery,Occur.MUST)
			.build();
	
	// 4. 指定索引库位置
	String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
	Directory directory = FSDirectory.open(Paths.get(indexDir));

	// 5. 打开索引库,并转换成:IndexReader
	IndexReader reader = DirectoryReader.open(directory);

	// 6. 创建搜索对象
	IndexSearcher searcher = new IndexSearcher(reader);

	// 7. 搜索并返回结果
	TopDocs topDocs = searcher.search(query, 1000);

	// 8. 转换结果集
	long totalHits = topDocs.totalHits;
	ScoreDoc[] docs = topDocs.scoreDocs;
	Optional<ScoreDoc[]> optional =  Optional.of(docs);
	optional.ifPresent((item)->{
		for(ScoreDoc doc : item) {
			// Lucene为Document分配的唯一ID,并非业务:pid
			int docID = doc.doc;
			try {
				Document document = reader.document(docID);
				System.out.println("==================================================");
				System.out.println("  pid: "+document.get("pid"));
				System.out.println("  pname: "+document.get("pname"));
				System.out.println("  price: "+document.get("price"));
			} catch (IOException e) {
			}
		}
	});
	
	System.err.println("totalHits:" + totalHits);
	// 9. 释放资源
	reader.close();
}
```
### (4). 总结
