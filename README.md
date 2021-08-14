# RDF misc

## VirtuosoのVALUESの要素数

VALUESの要素数が4095以上で、下記のエラー
```
Error: 400
Virtuoso 37000 Error SP030: SPARQL compiler, line 0: Too many arguments of a standard built-in function in operator()
```
Virtuosoソースコードの`libsrc/Wi/sparql_core.c`を見ると、`0xFFF`(=4095)とハードコーディングされている.

### 変更1
`libsrc/Wi/sparql_core.c`で以下の部分をコメントアウト
```
      if (argcount > sbd->sbd_maxargs)
        sparyyerror_impl (sparp, NULL, t_box_sprintf (100, "Too many arguments of a standard built-in function %s()", sbd->sbd_name));
```

1万要素とかでも普通に渡せるようになる.

### 変更2
要素数をさらに増やしていくと、以下のエラーに遭遇する.
```
Error: 400 Bad Request
Virtuoso 37000 Error SP031: SPARQL: Internal error: The length of generated SQL text has exceeded 10000 lines of code
```
`libsrc/Wi/sparql2sql.h`で以下の部分を削除
```
    if (SSG_MAX_ALLOWED_LINE_COUNT == ssg->ssg_line_count++) \
      spar_sqlprint_error_impl (ssg, "The length of generated SQL text has exceeded 10000 lines of code"); \
```
すると、さらに以下のエラーが出る.
```
Error: 500 Internal Server Error
Virtuoso ..... Error SQ200: Query too large, variables in state over the limit
```

### 変更3
`libsrc/Wi/wi.h`で以下の部分を修正
```
#ifdef LARGE_QI_INST
#define MAX_STATE_SLOTS 0xffffe
#else
#define MAX_STATE_SLOTS 0xfffe
#endif
```
`MAX_STATE_SLOTS`を`0xffffe`にする(16倍にする).

すると以下のエラーが出る.
```
Error: 500 Internal Server Error
Virtuoso 42000 Error SQ199: Maximum size (32767) of a code vector exceeded by 3967441 bytes. Please split the code in smaller units.
```

### 変更4
`libsrc/Wi/sqlexp.c`の以下の部分をコメントアウト
```
      if (BOFS_TO_OFS (byte_len) > SHRT_MAX)
	{
	  sqlc_new_error (sc->sc_cc, "42000", "SQ199",
	      "Maximum size (%ld) of a code vector exceeded by %ld bytes. "
	      "Please split the code in smaller units.", (long) SHRT_MAX, (long) (byte_len - SHRT_MAX));
	}
```
以上4カ所の修正で、結果的には、数万から10万要素でも答えが返ってくるようになる. ただし数万以上ともなると急激に遅くなり、何分もかかる.

# Virtuosoのデフォルト1万行問題

クエリの結果がResultSetMaxRows(デフォルト10000)で切れている場合は、レスポンスヘッダーにX-SPARQL-MaxRows: 10000と設定されるので、それを見てOFFSETをずらしていけば、結果がとれるはず。

`spang2` can obtain results from *Virtuoso* by **automatic pagenation**.

## Examples
Example SPARQL query:
```
# @endpoint https://www.orpha.net/sparql

PREFIX oboInOwl: <http://www.geneontology.org/formats/oboInOwl#>

SELECT ?s ?o
WHERE {
    ?s oboInOwl:hasDbXref ?o
}
```

Obtain data from SPARQL endpoint.
```
$ spang2 orphanet.rq > orphanet.tsv
Querying for the next page (OFFSET 10000 LIMIT 10000)...
Querying for the next page (OFFSET 20000 LIMIT 10000)...
Querying for the next page (OFFSET 30000 LIMIT 10000)...
Querying for the next page (OFFSET 40000 LIMIT 10000)...
$ wc orphanet.tsv
  42571   85142 3791019 orphanet.tsv
```

You can also execute the query file as follows, if the
file includes shebang line.
```
$ orphanet.rq > orphanet.tsv
```

Even without the query file, you can execute spang comand-line shortcuts equivalent to the query file.
```
$ spang2 -e orphanet -P oboInOwl:hasDbXref > orphanet.tsv
```

### Example usecae of the resulting tsv
Extract subset of tsv to make Turtle.
```
$ ./orphanet-ensembl.sh > orphanet-ensembl.ttl
$ grep -c ensembl orphanet-ensembl.ttl
3213
```

You can also use FILTER at the SPARQL level to extract the subset (see
`orphanet_filter.rq`).
