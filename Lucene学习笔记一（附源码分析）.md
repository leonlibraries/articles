---
title: Lucene 学习笔记一（附源码分析）
date: 2017-02-27 10:27:15
tags: [lucene,full-text search]
categories: 全文搜索
---
![全文搜索](full-text-searching.jpg)

# 背景介绍
>Apache Lucene is a high-performance, full-featured text search engine library written entirely in Java. It is a technology suitable for nearly any application that requires full-text search, especially cross-platform.

有几个关键点，全文搜索库（Library）、由 Java 编写以及跨平台。本文基于 Apache Lucene 6.4.1 版本来进行讨论。

# 基础概念
### **Document**
文档，由一系列的字段（field）组成，本质上就是 field(name->value) 的集合。无论是 PDF、WORD 还是 TXT 等数据源都需要实现``DocumentHandler``接口最终生成``Document.class``类的实例，而这个实例就是这里说的文档。

### **Inverted Index**
倒排索引，由一系列的『词—>document-ids』组成，如果 field 标记为不存储，那么在获取结果的过程中，是无法获取到字段信息的，而是需要通过全文搜索得到的 ID 序列，再去其他存储系统如关系型数据库中获取出来。

![倒排索引](invert-index-sample.jpg)

### **TokenStream**
属于 analysis 的范围，用于从文档的字段中以及查询中枚举出一系列的词（token），其有两个实现类``Tokenizer``和``TokenFilter``，以下是官方对此的解释：
> - Tokenizer, a TokenStream whose input is a Reader;
- TokenFilter, a TokenStream whose input is another TokenStream.

后续对此会有进一步研究。
# 关键算法流程
### 文档化 -> 分析 -> 建立索引的过程
![索引过程](document-analysis-index.png)

Lucene 只是负责 indexing 和 searching 阶段，至于如何将普通文件转化为 Document，这个交由开发者自行实现。

### 检索阶段
![检索过程](searching.png)

The Query Parser ： 剖析用户传入的查询 String（词、句子、一段话都有可能），并将分析后的结果传入 IndexSearcher；

The IndexSearcher ：包含 Analyzer、IndexReader、Scorer、Result Collector ，后续会对其进行进一步挖掘。

PS ：searching 过程中调用的 Analyzer 和 indexing 过程中的 Analyzer 需保持一致。

### 整体流程
![整体流程图](index-and-search.png)

可以看出 IndexWriter 和 QueryParser 出现共用 Analyzer 的情况，不同的索引的提词方式，有不同的检索效果。
# 基础演示

索引过程
```java
/**
 * Index all text files under a directory.
 * <p>
 * This is a command-line application demonstrating simple Lucene indexing.
 * Run it with no command-line arguments for usage information.
 */
public class IndexFiles {


    private IndexFiles() {
    }

    /**
     * Index all text files under a directory.
     */
    public static void main(String args[]) {
        String usage = "java org.wlr.lucene.learning.IndexFiles"
                + " [-index INDEX_PATH] [-docs DOCS_PATH] [-update]\n\n"
                + "This indexes the documents in DOCS_PATH, creating a Lucene index"
                + "in INDEX_PATH that can be searched with SearchFiles";
        String indexPath = "index";
        String docsPath = null;
        boolean create = true;

        for (int i = 0; i < args.length; i++) {
            if ("-index".equals(args[i])) {
                indexPath = args[i + 1];
                i++;
            } else if ("-docs".equals(args[i])) {
                docsPath = args[i + 1];
                i++;
            } else if ("-update".equals(args[i])) {
                create = false;
            }
        }

        if (docsPath == null) {
            System.err.println("Usage: " + usage);
            System.exit(1);
        }

        final Path docDir = Paths.get(docsPath);
        if (!Files.isReadable(docDir)) {
            System.out.println("Document directory '" + docDir.toAbsolutePath() + "' does not exist or is not readable, please check the path");
            System.exit(1);
        }

        Date start = new Date();
        try {
            System.out.println("Indexing to directory '" + indexPath + "'...");

            Directory dir = FSDirectory.open(Paths.get(indexPath));
            Analyzer analyzer = new StandardAnalyzer();
            IndexWriterConfig iwc = new IndexWriterConfig(analyzer);

            if (create) {
                // Create a new index in the directory, removing any
                // previously indexed documents:
                iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE);
            } else {
                // Add new documents to an existing index:
                iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE_OR_APPEND);
            }

            // Optional: for better indexing performance, if you
            // are indexing many documents, increase the RAM
            // buffer.  But if you do this, increase the max heap
            // size to the JVM (eg add -Xmx512m or -Xmx1g):
            //
            // iwc.setRAMBufferSizeMB(256.0);
            IndexWriter writer = new IndexWriter(dir, iwc);
            indexDocs(writer, docDir);

            // NOTE: if you want to maximize search performance,
            // you can optionally call forceMerge here.  This can be
            // a terribly costly operation, so generally it's only
            // worth it when your index is relatively static (ie
            // you're done adding documents to it):
            //
            // writer.forceMerge(1);

            writer.close();

            Date end = new Date();
            System.out.println(end.getTime() - start.getTime() + " total milliseconds");

        } catch (IOException e) {
            System.out.println(" caught a " + e.getClass() +
                    "\n with message: " + e.getMessage());
        }


    }

    /**
     * Indexes the given file using the given writer, or if a directory is given,
     * recurses over files and directories found under the given directory.
     * <p>
     * NOTE: This method indexes one document per input file.  This is slow.  For good
     * throughput, put multiple documents into your input file(s).  An example of this is
     * in the benchmark module, which can create "line doc" files, one document per line,
     * using the
     * <a href="../../../../../contrib-benchmark/org/apache/lucene/benchmark/byTask/tasks/WriteLineDocTask.html"
     * >WriteLineDocTask</a>.
     *
     * @param writer Writer to the index where the given file/dir info will be stored
     * @param path   The file to index, or the directory to recurse into to find files to index
     * @throws IOException If there is a low-level I/O error
     */
    static void indexDocs(final IndexWriter writer, Path path) throws IOException {
        if (Files.isDirectory(path)) {
            Files.walkFileTree(path, new SimpleFileVisitor<Path>() {
                @Override
                public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
                    try {
                        indexDoc(writer, file, attrs.lastModifiedTime().toMillis());
                    } catch (IOException ignore) {
                        //ignore
                    }
                    return FileVisitResult.CONTINUE;
                }
            });
        } else {
            indexDoc(writer, path, Files.getLastModifiedTime(path).toMillis());
        }

    }

    /**
     * Indexes a single document
     */
    static void indexDoc(IndexWriter writer, Path file, long lastModified) throws IOException {
        try (InputStream stream = Files.newInputStream(file)) {
            // make a new, empty document
            Document doc = new Document();

            // Add the path of the file as a field named "path".  Use a
            // field that is indexed (i.e. searchable), but don't tokenize
            // the field into separate words and don't index term frequency
            // or positional information:
            Field pathField = new StringField("path", file.toString(), Field.Store.YES);
            doc.add(pathField);

            // Add the last modified date of the file a field named "modified".
            // Use a LongPoint that is indexed (i.e. efficiently filterable with
            // PointRangeQuery).  This indexes to milli-second resolution, which
            // is often too fine.  You could instead create a number based on
            // year/month/day/hour/minutes/seconds, down the resolution you require.
            // For example the long value 2011021714 would mean
            // February 17, 2011, 2-3 PM.
            doc.add(new LongPoint("modified", lastModified));

            // Add the contents of the file to a field named "contents".  Specify a Reader,
            // so that the text of the file is tokenized and indexed, but not stored.
            // Note that FileReader expects the file to be in UTF-8 encoding.
            // If that's not the case searching for special characters will fail.
            doc.add(new TextField("contents", new BufferedReader(new InputStreamReader(stream, StandardCharsets.UTF_8))));

            // store 只管结果返回展示部分的情况，yes 可以展示出来，no 展示不出来，
            // StringField IndexOptions 为 DOCS，因此允许搜索，但不用分词器分词，
            // 当 Store 为 NO 的时候， 该字段可以匹配搜索但是展示不出来
            Field ext = new StringField("ext", "synchronized", Field.Store.YES);
            doc.add(ext);

            if (writer.getConfig().getOpenMode() == IndexWriterConfig.OpenMode.CREATE) {
                // New index, so we just add the document (no old document can be there):
                System.out.println("adding " + file);
                writer.addDocument(doc);
            } else {
                // Existing index (an old copy of this document may have been indexed) so
                // we use updateDocument instead to replace the old one matching the exact
                // path, if present:
                System.out.println("updating " + file);
                writer.updateDocument(new Term("path", file.toString()), doc);
            }
        }
    }
}

```

查询阶段
```java
/**
 * Simple command-line based search demo.
 */
public class SearchFiles {

    private SearchFiles() {
    }

    /**
     * Simple command-line based search demo.
     */
    public static void main(String[] args) throws Exception {
        String usage = "Usage:\tjava org.wlr.lucene.learning.SearchFiles [-index dir] [-field f] [-repeat n] [-queries file] [-query string] [-raw] [-paging hitsPerPage]\n\nSee http://lucene.apache.org/core/4_1_0/demo/ for details.";
        if (args.length > 0 && ("-h".equals(args[0])) || "-help".equals(args[0])) {
            System.out.println(usage);
            System.exit(0);
        }

        String index = "index";
        String field = "contents";
        String queries = null;
        int repeat = 0;
        boolean raw = false;
        String queryString = null;
        int hitsPerPage = 10;

        for (int i = 0; i < args.length; i++) {
            if ("-index".equals(args[i])) {
                index = args[i + 1];
                i++;
            } else if ("-field".equals(args[i])) {
                field = args[i + 1];
                i++;
            } else if ("-queries".equals(args[i])) {
                queries = args[i + 1];
                i++;
            } else if ("-query".equals(args[i])) {
                queryString = args[i + 1];
                i++;
            } else if ("-repeat".equals(args[i])) {
                repeat = Integer.parseInt(args[i + 1]);
                i++;
            } else if ("-raw".equals(args[i])) {
                raw = true;
            } else if ("-paging".equals(args[i])) {
                hitsPerPage = Integer.parseInt(args[i + 1]);
                if (hitsPerPage <= 0) {
                    System.err.println("There must be at least 1 hit per page.");
                    System.exit(1);
                }
                i++;
            }
        }

        IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(index)));
        IndexSearcher searcher = new IndexSearcher(reader);

        // 标准分词器
        Analyzer analyzer = new StandardAnalyzer();

        BufferedReader in = null;
        if (queries != null) {
            in = Files.newBufferedReader(Paths.get(queries), StandardCharsets.UTF_8);
        } else {
            in = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
        }

        QueryParser parser = new QueryParser(field, analyzer);
        QueryParser parserExt = new QueryParser("ext", analyzer);

        while (true) {
            if (queries == null && queryString == null) {
                System.out.println("Enter query: ");
            }

            String line = queryString != null ? queryString : in.readLine();

            if (line == null || line.length() == -1) {
                break;
            }

            line = line.trim();
            if (line.length() == 0) {
                break;
            }

            Query query = parser.parse(line);
            Query queryExt = parserExt.parse(line);

            // 多字段匹配
            BooleanQuery booleanQuery = new BooleanQuery.Builder().add(query, BooleanClause.Occur.SHOULD)
                    .add(queryExt, BooleanClause.Occur.SHOULD).build();

            System.out.println("Searching for: " + query.toString(field));

            if (repeat > 0) {                   // repeat & time as benchmark
                Date start = new Date();
                for (int i = 0; i < repeat; i++) {
                    searcher.search(booleanQuery, 100);
                }

                Date end = new Date();
                System.out.println("Time: " + (end.getTime() - start.getTime()) + "ms");
            }

            doPagingSearch(in, searcher, booleanQuery, hitsPerPage, raw, queries == null && queryString == null);

            if (queryString != null) {
                break;
            }
        }
        reader.close();
    }

    /**
     * This demonstrates a typical paging search scenario, where the search engine presents
     * pages of size n to the user. The user can then go to the next page if interested in
     * the next hits.
     * <p>
     * When the query is executed for the first time, then only enough results are collected
     * to fill 5 result pages. If the user wants to page beyond this limit, then the query
     * is executed another time and all hits are collected.
     */

    public static void doPagingSearch(BufferedReader in, IndexSearcher searcher, Query query,
                                      int hitsPerPage, boolean raw, boolean interactive) throws IOException {
        // Collect enough docs to show 5 pages
        TopDocs results = searcher.search(query, 5 * hitsPerPage);
        ScoreDoc[] hits = results.scoreDocs;

        int numTotalHits = results.totalHits;
        System.out.println(numTotalHits + " total matching documents");

        int start = 0;
        int end = Math.min(numTotalHits, hitsPerPage);

        while (true) {
            if (end > hits.length) {
                System.out.println("Only results 1 - " + hits.length + " of " + numTotalHits + " total matching documents collected.");
                System.out.println("Collect more (y/n) ?");
                String line = in.readLine();
                if (line.length() == 0 || line.charAt(0) == 'n') {
                    break;
                }

                hits = searcher.search(query, numTotalHits).scoreDocs;
            }

            end = Math.min(hits.length, start + hitsPerPage);

            for (int i = start; i < end; i++) {
                if (raw) {   // output raw format
                    System.out.println("doc=" + hits[i].doc + " score=" + hits[i].score);
                    continue;
                }

                Document doc = searcher.doc(hits[i].doc);
                String path = doc.get("path");

                if (path != null) {
                    System.out.println((i + 1) + ". " + path);
                    String title = doc.get("title");
                    if (title != null) {
                        System.out.println("   Title: " + doc.get("title"));
                    }
                } else {
                    System.out.println((i + 1) + ". " + "No path for this document");
                }
            }

            if (!interactive || end == 0) {
                break;
            }

            if (numTotalHits >= end) {
                boolean quit = false;
                while (true) {
                    System.out.print("Press ");
                    if (start - hitsPerPage >= 0) {
                        System.out.println("(p)revious page,");
                    }
                    if (start + hitsPerPage < numTotalHits) {
                        System.out.print("(n)ext page, ");
                    }
                    System.out.println("(q)uit or enter number to jump to a page.");

                    String line = in.readLine();
                    if (line.length() == 0 || line.charAt(0) == 'q') {
                        quit = true;
                        break;
                    }
                    if (line.charAt(0) == 'p') {
                        quit = true;
                        break;
                    } else if (line.charAt(0) == 'n') {
                        if (start + hitsPerPage < numTotalHits) {
                            start += hitsPerPage;
                        }
                        break;
                    } else {
                        int page = Integer.parseInt(line);
                        if ((page - 1) * hitsPerPage < numTotalHits) {
                            start = (page - 1) * hitsPerPage;
                            break;
                        } else {
                            System.out.println("No such page");
                        }
                    }

                }
                if (quit) break;
                end = Math.min(numTotalHits, start + hitsPerPage);
            }
        }
    }
}
```

# 源码分析

### Document
来看看源码，从 ``Document`` 看起 (这里Document类代码不完整)

```java

/** Documents are the unit of indexing and search.
 *
 * A Document is a set of fields.  Each field has a name and a textual value.
 * A field may be {@link org.apache.lucene.index.IndexableFieldType#stored() stored} with the document, in which
 * case it is returned with search hits on the document.  Thus each document
 * should typically contain one or more stored fields which uniquely identify
 * it.
 *
 * <p>Note that fields which are <i>not</i> {@link org.apache.lucene.index.IndexableFieldType#stored() stored} are
 * <i>not</i> available in documents retrieved from the index, e.g. with {@link
 * ScoreDoc#doc} or {@link IndexReader#document(int)}.
 */

public final class Document implements Iterable<IndexableField> {

  private final List<IndexableField> fields = new ArrayList<>();

  /** Constructs a new document with no fields. */
  public Document() {}

  @Override
  public Iterator<IndexableField> iterator() {
    return fields.iterator();
  }

  /**
   * <p>Adds a field to a document.  Several fields may be added with
   * the same name.  In this case, if the fields are indexed, their text is
   * treated as though appended for the purposes of search.</p>
   * <p> Note that add like the removeField(s) methods only makes sense
   * prior to adding a document to an index. These methods cannot
   * be used to change the content of an existing index! In order to achieve this,
   * a document has to be deleted from an index and a new changed version of that
   * document has to be added.</p>
   */
  public final void add(IndexableField field) {
    fields.add(field);
  }

  /**
   * <p>Removes field with the specified name from the document.
   * If multiple fields exist with this name, this method removes the first field that has been added.
   * If there is no field with the specified name, the document remains unchanged.</p>
   * <p> Note that the removeField(s) methods like the add method only make sense
   * prior to adding a document to an index. These methods cannot
   * be used to change the content of an existing index! In order to achieve this,
   * a document has to be deleted from an index and a new changed version of that
   * document has to be added.</p>
   */
  public final void removeField(String name) {
    Iterator<IndexableField> it = fields.iterator();
    while (it.hasNext()) {
      IndexableField field = it.next();
      if (field.name().equals(name)) {
        it.remove();
        return;
      }
    }
  }

  /**
   * <p>Removes all fields with the given name from the document.
   * If there is no field with the specified name, the document remains unchanged.</p>
   * <p> Note that the removeField(s) methods like the add method only make sense
   * prior to adding a document to an index. These methods cannot
   * be used to change the content of an existing index! In order to achieve this,
   * a document has to be deleted from an index and a new changed version of that
   * document has to be added.</p>
   */
  public final void removeFields(String name) {
    Iterator<IndexableField> it = fields.iterator();
    while (it.hasNext()) {
      IndexableField field = it.next();
      if (field.name().equals(name)) {
        it.remove();
      }
    }
  }

  /** Returns a field with the given name if any exist in this document, or
   * null.  If multiple fields exists with this name, this method returns the
   * first value added.
   */
  public final IndexableField getField(String name) {
    for (IndexableField field : fields) {
      if (field.name().equals(name)) {
        return field;
      }
    }
    return null;
  }

  /**
   * Returns an array of {@link IndexableField}s with the given name.
   * This method returns an empty array when there are no
   * matching fields.  It never returns null.
   *
   * @param name the name of the field
   * @return a <code>Field[]</code> array
   */
  public IndexableField[] getFields(String name) {
    List<IndexableField> result = new ArrayList<>();
    for (IndexableField field : fields) {
      if (field.name().equals(name)) {
        result.add(field);
      }
    }

    return result.toArray(new IndexableField[result.size()]);
  }

  /** Returns a List of all the fields in a document.
   * <p>Note that fields which are <i>not</i> stored are
   * <i>not</i> available in documents retrieved from the
   * index, e.g. {@link IndexSearcher#doc(int)} or {@link
   * IndexReader#document(int)}.
   *
   * @return an immutable <code>List&lt;Field&gt;</code>
   */
  public final List<IndexableField> getFields() {
    return Collections.unmodifiableList(fields);
  }

  private final static String[] NO_STRINGS = new String[0];

  /**
   * Returns an array of values of the field specified as the method parameter.
   * This method returns an empty array when there are no
   * matching fields.  It never returns null.
   * For a numeric {@link StoredField} it returns the string value of the number. If you want
   * the actual numeric field instances back, use {@link #getFields}.
   * @param name the name of the field
   * @return a <code>String[]</code> of field values
   */
  public final String[] getValues(String name) {
    List<String> result = new ArrayList<>();
    for (IndexableField field : fields) {
      if (field.name().equals(name) && field.stringValue() != null) {
        result.add(field.stringValue());
      }
    }

    if (result.size() == 0) {
      return NO_STRINGS;
    }

    return result.toArray(new String[result.size()]);
  }

  /** Returns the string value of the field with the given name if any exist in
   * this document, or null.  If multiple fields exist with this name, this
   * method returns the first value added. If only binary fields with this name
   * exist, returns null.
   * For a numeric {@link StoredField} it returns the string value of the number. If you want
   * the actual numeric field instance back, use {@link #getField}.
   */
  public final String get(String name) {
    for (IndexableField field : fields) {
      if (field.name().equals(name) && field.stringValue() != null) {
        return field.stringValue();
      }
    }
    return null;
  }

  /** Removes all the fields from document. */
  public void clear() {
    fields.clear();
  }
}
```
我们可以看到 ``Document`` 实现了 ``Iterable<IndexableField>``接口，也就意味着 ``Document`` 本身是有多个 ``IndexableField`` 组成的。

### DefaultIndexingChain

创建 ``DocConsumer`` 消费文档的内容
```java
   // this should be the last call in the ctor
   // it really sucks that we need to pull this within the ctor and pass this ref to the chain!
   DocConsumer consumer = indexWriterConfig.getIndexingChain().getChain(this);
```
其中创建的有可能是默认的索引处理链 ，``DocConsumer``  的子类``DefaultIndexingChain``，该类包含了将文档转化为索引的核心逻辑，十分重要。后续将从此类展开来讲。


``DefaultIndexingChain``遍历文档里的每一个字段(field)
```java
      for (IndexableField field : docState.doc) {
        fieldCount = processField(field, fieldGen, fieldCount);
      }
```
接下来着重看下``processField``方法，这里包含了一个字段是否被索引、stored、以及 DocValue 存储的核心逻辑
```java
private int processField(IndexableField field, long fieldGen, int fieldCount) throws IOException, AbortingException {
    String fieldName = field.name();
    IndexableFieldType fieldType = field.fieldType();

    PerField fp = null;

    if (fieldType.indexOptions() == null) {
      throw new NullPointerException("IndexOptions must not be null (field: \"" + field.name() + "\")");
    }

    // Invert indexed fields:
    if (fieldType.indexOptions() != IndexOptions.NONE) {

      // if the field omits norms, the boost cannot be indexed.
      if (fieldType.omitNorms() && field.boost() != 1.0f) {
        throw new UnsupportedOperationException("You cannot set an index-time boost: norms are omitted for field '" + field.name() + "'");
      }

      fp = getOrAddField(fieldName, fieldType, true);
      boolean first = fp.fieldGen != fieldGen;
      fp.invert(field, first);

      if (first) {
        fields[fieldCount++] = fp;
        fp.fieldGen = fieldGen;
      }
    } else {
      verifyUnIndexedFieldType(fieldName, fieldType);
    }

    // Add stored fields:
    if (fieldType.stored()) {
      if (fp == null) {
        fp = getOrAddField(fieldName, fieldType, false);
      }
      if (fieldType.stored()) {
        String value = field.stringValue();
        if (value != null && value.length() > IndexWriter.MAX_STORED_STRING_LENGTH) {
          throw new IllegalArgumentException("stored field \"" + field.name() + "\" is too large (" + value.length() + " characters) to store");
        }
        try {
          storedFieldsWriter.writeField(fp.fieldInfo, field);
        } catch (Throwable th) {
          throw AbortingException.wrap(th);
        }
      }
    }

    DocValuesType dvType = fieldType.docValuesType();
    if (dvType == null) {
      throw new NullPointerException("docValuesType must not be null (field: \"" + fieldName + "\")");
    }
    if (dvType != DocValuesType.NONE) {
      if (fp == null) {
        fp = getOrAddField(fieldName, fieldType, false);
      }
      indexDocValue(fp, dvType, field);
    }
    if (fieldType.pointDimensionCount() != 0) {
      if (fp == null) {
        fp = getOrAddField(fieldName, fieldType, false);
      }
      indexPoint(fp, field);
    }

    return fieldCount;
  }
```
进入 ``getOrAddField`` 方法
```java
    // fieldHash 维护了一套字段集合 ，通过名字如果能在其 Segment 中找到其对应的字段，那么就不必要重新建立新字段了
    PerField fp = fieldHash[hashPos];
    while (fp != null && !fp.fieldInfo.name.equals(name)) {
      fp = fp.next;
    }

    // 如果在所属 segment 下找不到对应的字段，那么就要考虑新建了
    if (fp == null) {
      // First time we are seeing this field in this segment

      FieldInfo fi = fieldInfos.getOrAdd(name);
      // Messy: must set this here because e.g. FreqProxTermsWriterPerField looks at the initial
      // IndexOptions to decide what arrays it must create).  Then, we also must set it in
      // PerField.invert to allow for later downgrading of the index options:
      fi.setIndexOptions(fieldType.indexOptions());

      fp = new PerField(fi, invert);
      fp.next = fieldHash[hashPos];
      fieldHash[hashPos] = fp;
      totalFieldCount++;

      // At most 50% load factor:
      if (totalFieldCount >= fieldHash.length/2) {
        // 字段超过占用50%则需要做 rehash 的动作，目的是扩充 fieldHash 数组大小
        rehash();
      }

      if (totalFieldCount > fields.length) {
        // 扩充 fields 数组大小，fields 数组是用来维护一个 document 里所有的字段信息的
        PerField[] newFields = new PerField[ArrayUtil.oversize(totalFieldCount, RamUsageEstimator.NUM_BYTES_OBJECT_REF)];
        System.arraycopy(fields, 0, newFields, 0, fields.length);
        fields = newFields;
      }

    } else if (invert && fp.invertState == null) {
      // Messy: must set this here because e.g. FreqProxTermsWriterPerField looks at the initial
      // IndexOptions to decide what arrays it must create).  Then, we also must set it in
      // PerField.invert to allow for later downgrading of the index options:
      fp.fieldInfo.setIndexOptions(fieldType.indexOptions());
      fp.setInvertState();
    }
```
关于 ``fieldHash`` 数组：每一个元素表示为一个 ``Segment``，每个 ``Segment`` 中以链表的形式维护一个``Field``序列。
```java
      fp = getOrAddField(fieldName, fieldType, true);
      boolean first = fp.fieldGen != fieldGen;
      fp.invert(field, first);

      if (first) {
        fields[fieldCount++] = fp;
        fp.fieldGen = fieldGen;
      }
```
``fieldGen``用来标记字段是否是第一次出现，如果是第一次出现，那么需要将字段放到 ``fields``数组中维护。

### TokenStream

接下来继续看看建立倒排索引最核心的方法 ``fp.invert(field, first);``
```java
    /** Inverts one field for one document; first is true
     *  if this is the first time we are seeing this field
     *  name in this document. */
    public void invert(IndexableField field, boolean first) throws IOException, AbortingException {
      if (first) {
        // First time we're seeing this field (indexed) in
        // this document:
        invertState.reset();
      }

      IndexableFieldType fieldType = field.fieldType();

      IndexOptions indexOptions = fieldType.indexOptions();
      fieldInfo.setIndexOptions(indexOptions);

      if (fieldType.omitNorms()) {
        fieldInfo.setOmitsNorms();
      }

      final boolean analyzed = fieldType.tokenized() && docState.analyzer != null;

      // only bother checking offsets if something will consume them.
      // TODO: after we fix analyzers, also check if termVectorOffsets will be indexed.
      final boolean checkOffsets = indexOptions == IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS;

      /*
       * To assist people in tracking down problems in analysis components, we wish to write the field name to the infostream
       * when we fail. We expect some caller to eventually deal with the real exception, so we don't want any 'catch' clauses,
       * but rather a finally that takes note of the problem.
       */
      boolean succeededInProcessingField = false;
      try (TokenStream stream = tokenStream = field.tokenStream(docState.analyzer, tokenStream)) {
        // reset the TokenStream to the first token
        stream.reset();
        invertState.setAttributeSource(stream);
        termsHashPerField.start(field, first);

        while (stream.incrementToken()) {

          // If we hit an exception in stream.next below
          // (which is fairly common, e.g. if analyzer
          // chokes on a given document), then it's
          // non-aborting and (above) this one document
          // will be marked as deleted, but still
          // consume a docID

          int posIncr = invertState.posIncrAttribute.getPositionIncrement();
          invertState.position += posIncr;
          if (invertState.position < invertState.lastPosition) {
            if (posIncr == 0) {
              throw new IllegalArgumentException("first position increment must be > 0 (got 0) for field '" + field.name() + "'");
            } else if (posIncr < 0) {
              throw new IllegalArgumentException("position increment must be >= 0 (got " + posIncr + ") for field '" + field.name() + "'");
            } else {
              throw new IllegalArgumentException("position overflowed Integer.MAX_VALUE (got posIncr=" + posIncr + " lastPosition=" + invertState.lastPosition + " position=" + invertState.position + ") for field '" + field.name() + "'");
            }
          } else if (invertState.position > IndexWriter.MAX_POSITION) {
            throw new IllegalArgumentException("position " + invertState.position + " is too large for field '" + field.name() + "': max allowed position is " + IndexWriter.MAX_POSITION);
          }
          invertState.lastPosition = invertState.position;
          if (posIncr == 0) {
            invertState.numOverlap++;
          }

          if (checkOffsets) {
            int startOffset = invertState.offset + invertState.offsetAttribute.startOffset();
            int endOffset = invertState.offset + invertState.offsetAttribute.endOffset();
            if (startOffset < invertState.lastStartOffset || endOffset < startOffset) {
              throw new IllegalArgumentException("startOffset must be non-negative, and endOffset must be >= startOffset, and offsets must not go backwards "
                                                 + "startOffset=" + startOffset + ",endOffset=" + endOffset + ",lastStartOffset=" + invertState.lastStartOffset + " for field '" + field.name() + "'");
            }
            invertState.lastStartOffset = startOffset;
          }

          invertState.length++;
          if (invertState.length < 0) {
            throw new IllegalArgumentException("too many tokens in field '" + field.name() + "'");
          }
          //System.out.println("  term=" + invertState.termAttribute);

          // If we hit an exception in here, we abort
          // all buffered documents since the last
          // flush, on the likelihood that the
          // internal state of the terms hash is now
          // corrupt and should not be flushed to a
          // new segment:
          try {
            termsHashPerField.add();
          } catch (MaxBytesLengthExceededException e) {
            byte[] prefix = new byte[30];
            BytesRef bigTerm = invertState.termAttribute.getBytesRef();
            System.arraycopy(bigTerm.bytes, bigTerm.offset, prefix, 0, 30);
            String msg = "Document contains at least one immense term in field=\"" + fieldInfo.name + "\" (whose UTF8 encoding is longer than the max length " + DocumentsWriterPerThread.MAX_TERM_LENGTH_UTF8 + "), all of which were skipped.  Please correct the analyzer to not produce such terms.  The prefix of the first immense term is: '" + Arrays.toString(prefix) + "...', original message: " + e.getMessage();
            if (docState.infoStream.isEnabled("IW")) {
              docState.infoStream.message("IW", "ERROR: " + msg);
            }
            // Document will be deleted above:
            throw new IllegalArgumentException(msg, e);
          } catch (Throwable th) {
            throw AbortingException.wrap(th);
          }
        }

        // trigger streams to perform end-of-stream operations
        stream.end();

        // TODO: maybe add some safety? then again, it's already checked
        // when we come back around to the field...
        invertState.position += invertState.posIncrAttribute.getPositionIncrement();
        invertState.offset += invertState.offsetAttribute.endOffset();

        /* if there is an exception coming through, we won't set this to true here:*/
        succeededInProcessingField = true;
      } finally {
        if (!succeededInProcessingField && docState.infoStream.isEnabled("DW")) {
          docState.infoStream.message("DW", "An exception was thrown while processing field " + fieldInfo.name);
        }
      }

      if (analyzed) {
        invertState.position += docState.analyzer.getPositionIncrementGap(fieldInfo.name);
        invertState.offset += docState.analyzer.getOffsetGap(fieldInfo.name);
      }

      invertState.boost *= field.boost();
    }
```

我们大致可以看出，这里的逻辑无外乎根据索引选项``IndexOptions``来判断索引的颗粒度大小，进而再用``TokenStream``将字段里的内容进行分词、提词。每一个``Field``都有其对应的``TokenStream``。以下是``TokenStream``的源码参照：

```java
/**
 * A <code>TokenStream</code> enumerates the sequence of tokens, either from
 * {@link Field}s of a {@link Document} or from query text.
 * <p>
 * This is an abstract class; concrete subclasses are:
 * <ul>
 * <li>{@link Tokenizer}, a <code>TokenStream</code> whose input is a Reader; and
 * <li>{@link TokenFilter}, a <code>TokenStream</code> whose input is another
 * <code>TokenStream</code>.
 * </ul>
 * A new <code>TokenStream</code> API has been introduced with Lucene 2.9. This API
 * has moved from being {@link Token}-based to {@link Attribute}-based. While
 * {@link Token} still exists in 2.9 as a convenience class, the preferred way
 * to store the information of a {@link Token} is to use {@link AttributeImpl}s.
 * <p>
 * <code>TokenStream</code> now extends {@link AttributeSource}, which provides
 * access to all of the token {@link Attribute}s for the <code>TokenStream</code>.
 * Note that only one instance per {@link AttributeImpl} is created and reused
 * for every token. This approach reduces object creation and allows local
 * caching of references to the {@link AttributeImpl}s. See
 * {@link #incrementToken()} for further details.
 * <p>
 * <b>The workflow of the new <code>TokenStream</code> API is as follows:</b>
 * <ol>
 * <li>Instantiation of <code>TokenStream</code>/{@link TokenFilter}s which add/get
 * attributes to/from the {@link AttributeSource}.
 * <li>The consumer calls {@link TokenStream#reset()}.
 * <li>The consumer retrieves attributes from the stream and stores local
 * references to all attributes it wants to access.
 * <li>The consumer calls {@link #incrementToken()} until it returns false
 * consuming the attributes after each call.
 * <li>The consumer calls {@link #end()} so that any end-of-stream operations
 * can be performed.
 * <li>The consumer calls {@link #close()} to release any resource when finished
 * using the <code>TokenStream</code>.
 * </ol>
 * To make sure that filters and consumers know which attributes are available,
 * the attributes must be added during instantiation. Filters and consumers are
 * not required to check for availability of attributes in
 * {@link #incrementToken()}.
 * <p>
 * You can find some example code for the new API in the analysis package level
 * Javadoc.
 * <p>
 * Sometimes it is desirable to capture a current state of a <code>TokenStream</code>,
 * e.g., for buffering purposes (see {@link CachingTokenFilter},
 * TeeSinkTokenFilter). For this usecase
 * {@link AttributeSource#captureState} and {@link AttributeSource#restoreState}
 * can be used.
 * <p>The {@code TokenStream}-API in Lucene is based on the decorator pattern.
 * Therefore all non-abstract subclasses must be final or have at least a final
 * implementation of {@link #incrementToken}! This is checked when Java
 * assertions are enabled.
 */
public abstract class TokenStream extends AttributeSource implements Closeable {

  /** Default {@link AttributeFactory} instance that should be used for TokenStreams. */
  public static final AttributeFactory DEFAULT_TOKEN_ATTRIBUTE_FACTORY =
    AttributeFactory.getStaticImplementation(AttributeFactory.DEFAULT_ATTRIBUTE_FACTORY, PackedTokenAttributeImpl.class);

  /**
   * A TokenStream using the default attribute factory.
   */
  protected TokenStream() {
    super(DEFAULT_TOKEN_ATTRIBUTE_FACTORY);
    assert assertFinal();
  }

  /**
   * A TokenStream that uses the same attributes as the supplied one.
   */
  protected TokenStream(AttributeSource input) {
    super(input);
    assert assertFinal();
  }

  /**
   * A TokenStream using the supplied AttributeFactory for creating new {@link Attribute} instances.
   */
  protected TokenStream(AttributeFactory factory) {
    super(factory);
    assert assertFinal();
  }

  private boolean assertFinal() {
    try {
      final Class<?> clazz = getClass();
      if (!clazz.desiredAssertionStatus())
        return true;
      assert clazz.isAnonymousClass() ||
        (clazz.getModifiers() & (Modifier.FINAL | Modifier.PRIVATE)) != 0 ||
        Modifier.isFinal(clazz.getMethod("incrementToken").getModifiers()) :
        "TokenStream implementation classes or at least their incrementToken() implementation must be final";
      return true;
    } catch (NoSuchMethodException nsme) {
      return false;
    }
  }

  /**
   * Consumers (i.e., {@link IndexWriter}) use this method to advance the stream to
   * the next token. Implementing classes must implement this method and update
   * the appropriate {@link AttributeImpl}s with the attributes of the next
   * token.
   * <P>
   * The producer must make no assumptions about the attributes after the method
   * has been returned: the caller may arbitrarily change it. If the producer
   * needs to preserve the state for subsequent calls, it can use
   * {@link #captureState} to create a copy of the current attribute state.
   * <p>
   * This method is called for every token of a document, so an efficient
   * implementation is crucial for good performance. To avoid calls to
   * {@link #addAttribute(Class)} and {@link #getAttribute(Class)},
   * references to all {@link AttributeImpl}s that this stream uses should be
   * retrieved during instantiation.
   * <p>
   * To ensure that filters and consumers know which attributes are available,
   * the attributes must be added during instantiation. Filters and consumers
   * are not required to check for availability of attributes in
   * {@link #incrementToken()}.
   *
   * @return false for end of stream; true otherwise
   */
  public abstract boolean incrementToken() throws IOException;

  /**
   * This method is called by the consumer after the last token has been
   * consumed, after {@link #incrementToken()} returned <code>false</code>
   * (using the new <code>TokenStream</code> API). Streams implementing the old API
   * should upgrade to use this feature.
   * <p>
   * This method can be used to perform any end-of-stream operations, such as
   * setting the final offset of a stream. The final offset of a stream might
   * differ from the offset of the last token eg in case one or more whitespaces
   * followed after the last token, but a WhitespaceTokenizer was used.
   * <p>
   * Additionally any skipped positions (such as those removed by a stopfilter)
   * can be applied to the position increment, or any adjustment of other
   * attributes where the end-of-stream value may be important.
   * <p>
   * If you override this method, always call {@code super.end()}.
   *
   * @throws IOException If an I/O error occurs
   */
  public void end() throws IOException {
    endAttributes(); // LUCENE-3849: don't consume dirty atts
  }

  /**
   * This method is called by a consumer before it begins consumption using
   * {@link #incrementToken()}.
   * <p>
   * Resets this stream to a clean state. Stateful implementations must implement
   * this method so that they can be reused, just as if they had been created fresh.
   * <p>
   * If you override this method, always call {@code super.reset()}, otherwise
   * some internal state will not be correctly reset (e.g., {@link Tokenizer} will
   * throw {@link IllegalStateException} on further usage).
   */
  public void reset() throws IOException {}

  /** Releases resources associated with this stream.
   * <p>
   * If you override this method, always call {@code super.close()}, otherwise
   * some internal state will not be correctly reset (e.g., {@link Tokenizer} will
   * throw {@link IllegalStateException} on reuse).
   */
  @Override
  public void close() throws IOException {}

}
```

实际上，每一个``TokenStream``之间相互协调，通过``incrementToken()``方法串成了一条责任链。

以上面 sample 为例，在索引 ``content`` 字段的时候 TokenStream 链路是这样的：
``StandardTokenizer`` —> ``StandardFilter`` —> ``LowerCaseFilter`` —> ``FilteringTokenFilter``

Tokenizer 作为 Stream 的起点，直接通过``java.io.Reader``读取文件流数据，而其余``TokenFilter``则从其他的``TokenStream``中读取数据。

我们现在来看看链路的起点``StandardTokenizer``

```java
  /** A private instance of the JFlex-constructed scanner */
  private StandardTokenizerImpl scanner;

  private void init() {
    this.scanner = new StandardTokenizerImpl(input);
  }
  // this tokenizer generates three attributes:
  // term offset, positionIncrement and type
  private final CharTermAttribute termAtt = addAttribute(CharTermAttribute.class);
  private final OffsetAttribute offsetAtt = addAttribute(OffsetAttribute.class);
  private final PositionIncrementAttribute posIncrAtt = addAttribute(PositionIncrementAttribute.class);
  private final TypeAttribute typeAtt = addAttribute(TypeAttribute.class);
  /*
   * (non-Javadoc)
   *
   * @see org.apache.lucene.analysis.TokenStream#next()
   */
  @Override
  public final boolean incrementToken() throws IOException {
    clearAttributes();
    skippedPositions = 0;

    while(true) {
      int tokenType = scanner.getNextToken();

      if (tokenType == StandardTokenizerImpl.YYEOF) {
        return false;
      }

      if (scanner.yylength() <= maxTokenLength) {
        posIncrAtt.setPositionIncrement(skippedPositions+1);
        scanner.getText(termAtt);
        final int start = scanner.yychar();
        offsetAtt.setOffset(correctOffset(start), correctOffset(start+termAtt.length()));
        typeAtt.setType(StandardTokenizer.TOKEN_TYPES[tokenType]);
        return true;
      } else
        // When we skip a too-long term, we still increment the
        // position increment
        skippedPositions++;
    }
  }
```
``scanner.getText(termAtt);``实际上就是根据 Word Break 规则将词提取出来，而后续的``TokenFilter``只是对提取出来的 token 再次加工。

### Token 数据结构
一个 Token 包含 Offset Attribute , Term Attribute 以及 Type Attribute 三个元素
![Token 数据结构](token.png)
这个在 ``StandardTokenizer:incrementToken()``方法的源码中也是有体现的
