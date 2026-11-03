## 前言

页脚版权如果写死在 Java 字符串里，改一句文案就要翻接口实现，发版才生效。把两段模板挪到配置文件，需要替换的位置留格式化占位符，改文案只动配置。

```yaml
site:
  footer:
    original: 原文首发于示例站，转载请保留本链接。
    reprint: 本文转载自 %s，著作权归 %s。若链接失效请通知站点管理员。
```

`%s` 是 `Formatter` 的字符串占位，顺序与后面的参数一致。大写 `%S` 会把替换结果转成大写，文案里一般不要用它。

```java
String pattern = footerProperties.getReprint();
String line = String.format(pattern, article.getSourceName(), article.getWriter());
article.setFooter(line);
```

`String.format` 的第一个参数是模板，后面的变参按出现顺序填进 `%s`。参数个数少于占位符会抛 `MissingFormatArgumentException`，所以模板和调用点要一起改。转载来源或作者为空时先换成明确的“佚名”，不要把 `null` 直接交进去，否则页脚会出现字面量 `null`。
