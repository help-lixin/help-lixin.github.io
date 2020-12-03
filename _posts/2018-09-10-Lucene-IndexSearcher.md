---
layout: post
title: 'Lucene IndexSearcher简单检索(四)'
date: 2018-09-10
author: 李新
tags: Lucene
---


### (1). IndexSearcher搜索
```
package help.lixin.lucene.service;

import java.io.IOException;
import java.nio.file.Paths;
import java.util.Optional;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexReader;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.junit.Test;
import static junit.framework.Assert.*;

public class IndexSearchTest {

	@Test
	public void testSearch() throws Exception {
		// 1.创建分词器(写入时需要分词器,在搜索时:也是需要分词器对用户输入的内容进行分词的)
		// 搜索时的:分词器要和创建时使用的分词器也要一模一样的.
		Analyzer analyzer = new StandardAnalyzer();

		// 2.创建查询对象
		QueryParser queryParser = new QueryParser("pname", analyzer);

		// 3.设置搜索关键词
		// pname:手机
		// catalogName:手机
		Query query = queryParser.parse("手机");

		// 4. 指定索引库位置
		String indexDir = "/Users/lixin/Workspace/lucene-demo/indexDir";
		Directory directory = FSDirectory.open(Paths.get(indexDir));

		// 5. 打开索引库,并转换成:IndexReader
		IndexReader reader = DirectoryReader.open(directory);

		// 6. 创建搜索对象
		IndexSearcher searcher = new IndexSearcher(reader);

		// 7. 搜索并返回结果
		TopDocs topDocs = searcher.search(query, 10);

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
		
		assertEquals(364, totalHits);
		
		// 9. 释放资源
		reader.close();
	}
}
```
### (2). 总结
