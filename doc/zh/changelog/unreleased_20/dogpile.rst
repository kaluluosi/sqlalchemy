.. change::
    :tags: bug, examples

    dogpile_caching示例已针对2.0样式查询进行了更新。
    在“缓存查询”逻辑内部，添加了一个条件来区分执行失效操作时的“Query”和“select()”。