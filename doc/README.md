
# 翻译说明
## sphinx插件
需要安装sqlalchemy自己定制的三个插件
https://github.com/sqlalchemyorg/changelog.git
https://github.com/sqlalchemyorg/sphinx-paramlinks.git
https://github.com/sqlalchemyorg/zzzeeksphinx.git

## conf.py
```
extensions = [
    "sphinx.ext.autodoc",
    # "zzzeeksphinx", # # 要注释掉 这个插件
    # "changelog", # 要注释掉 这个插件
    # "sphinx_paramlinks", # 要注释掉 这个插件
    "sphinx_copybutton",
]
```

## 构建文档

gh-depploy.bat