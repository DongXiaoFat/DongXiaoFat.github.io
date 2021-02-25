---
title: 深入理解Android Font 机制
author: DongXiaoFat
date: 2021-02-16 16:30:00 +0800
categories: [Android, Framework]
tags: [Framework]
pin: true
---


# 深入理解Android Font 机制
> Android 字体是由配置好的ttf字体文件来描述的，然而在字体加载的过程中首先就是根据字符去匹配相应的ttf文件，然后在该ttf文件中依照Unicode编码查找字符形状进行描绘。

## Unicode编码

Unicode 规范规定，使用U+前缀加上一个十六进制的整数表示一个字符，比如U+0041表示大写拉丁字母A。而整个Unicode的字符集，需要U+000000到U+10FFFF的存储空间，一共使用了21bit共有17*2^16个位置。从U+000000到U+10FFFF,unicod的编码空间可以被划分为17个平面（plane），每个平面包含2^16个码位。17个平面的码位就可以表示为U+xx0000到U+xxFFFF,其中xx表示0x00-0X10,共17个平面。第一个平面称之基本多语言平面（Basic Multilingual Plane BMP），或称为第0平面。其他平面成为辅助平面（Supplement Planes）,其中，BMP平面内从U+D800-U+DFFF之间的码位为永久保留不映射到Unicode字符。其中UTF-16就利用该码位来对辅助平面的字符码位进行编码，目前Unicode官方支持的编码方式有三种 UTF-8,UTF-16,UTF-32,很多地方默认是UTF-16编码格式，无论是哪种编码其对应的Unicode码是唯一的。 

### UTF-8实现原理

UTF-8的规则如下：
- 单字节的字符，字节的第一位设置为0，后面7位为这个符号的Unicode码。因此对于英文字母，UTF-8编码和ASCII码相同（前127个基本的ASCII码不含剩下128扩展部分）。
- 对于n字节（n=2,3,4）的字符，第一个字节的前n位都设为1，第n+1位设为0，后面字节的前两位一律设为10.剩下的没有提及的二进制位，全部为这个符号的Unicode码。

|字节数|最小值|最大值|Byte1|Byte2|
|-----|------|-----|------|----|
|1|U+0000|U+007F|0xxxxxxx|-|
|2|U+7F+1=U+80|u+07FF|110xxxxx|10xxxxxx|
|3|U+7FF+1=U800|u+FFFF|1110xxxx|10xxxxxx|

例如：汉字的“汉”其对应的Unicode码是U+6C49,把6C49转成二进制是“0110 1100 0100 1001”大于u+07FF小于u+FFFF所以是3个字节，把上面二进制填写到对应的‘x'就得到“1110  0110 ，10 110001， 10 001001”所以“汉”对应的UTF8就是“0xE6 0xB1 0x89”

### UTF-16实现原理
Unicode的编码空间从U+000000到U+10FFFF,共分为17个平面，每个平面范围都是U+0000到U+FFFF,而该区间内的U+D800到U+DFFF是用于代理区域（Surrogate area），这个区域的码位是永久保留不映射到Unicode字符。UTF-16就利用保留下来的0xD800-0xDFFF区段的码位来对第1平面的字符（U+10000到U+10FFFF）的码位进行编码。辅助平面中的码位，在UTF-16中被编码为32位，4个字节。

- 编码算法

低于16位字符，每8位当作一个码元来表示UTF-16编码格式。

高于16位的字符如下：

辅助平面的码位（U+010000到U+10FFFF）减去0x10000,得到的值的范围就U+0到U+FFFFF最多占20位，

高位的10bit的值（0-0x3FF）被加上0XD800得到第一个码元，称为高位代理（high surrogate），值的范围0xD800到0xDBFF。

低位的10Bit的值（0-0x3FF）被加上0xDC00得到第二个码元，称作为低位代理（low surrogate）,值的范围0xDC00到0xDFFF。

低于16位的以汉字的“汉”来举例，6C49结果为 0x6C,0X49.
高于16位的如埃及象形文字“𓅘”其Unicode码为U+13158，我们减去0x10000后得到0x3158,二进制为“0011 0001 0101 1000”

区分它的高10位跟低10位：“0000 0011 00”跟“01 0101 1000”
- 添加0xD800到高10位，0xD800 +0xC = 0xD80C
- 添加0xDC00到低10位，0xDC00 + 0x158 = 0xDD58

如果按照大端在前也就是高位在前的顺序就是 ‘0xD80C 0xDD58’,反之小端在前就是‘0xDD58 0xD80C’。

UTF-16 BE：0xD80C 0xDD58
UTF-16 LE：0xDD58 0xD80C

## 字库文件加载

1. Android的ttf字体文件是放在system/fonts/ 目录下，而我们记录ttf的xml文件是防止在了system/etc/目录下。我们配置字体的时候可以通过 mk里的“PRODUCT_COPY_FILES”命令将对应的文件copy到相应的目录下。

2. 修改Android字体加载的模块，代码位于graphics库下，主要是修改xml文件

```java
// frameworks/base/graphics/java/android/graphics/Typeface.java

public static void buildSystemFallback(String xmlPath, String fontDir,
        ArrayMap<String, Typeface> fontMap, ArrayMap<String, FontFamily[]> fallbackMap) {
    try {
        //解析fonts.xml文件把内容放到FontConfig类中
        final FileInputStream fontsIn = new FileInputStream(xmlPath);
        final FontConfig fontConfig = FontListParser.parse(fontsIn);

        final HashMap<String, ByteBuffer> bufferCache = new HashMap<String, ByteBuffer>();
        final FontConfig.Family[] xmlFamilies = fontConfig.getFamilies();

        final ArrayMap<String, ArrayList<FontFamily>> fallbackListMap = new ArrayMap<>();
        //遍历出有名字的FontFamily防止到fallbackListMap。
        // First traverse families which have a 'name' attribute to create fallback map.
        for (final FontConfig.Family xmlFamily : xmlFamilies) {
            final String familyName = xmlFamily.getName();
            if (familyName == null) {
                continue;
            }
            final FontFamily family = createFontFamily(
                    xmlFamily.getName(), Arrays.asList(xmlFamily.getFonts()),
                    xmlFamily.getLanguages(), xmlFamily.getVariant(), bufferCache, fontDir);
            if (family == null) {
                continue;
            }
            final ArrayList<FontFamily> fallback = new ArrayList<>();
            fallback.add(family);
            fallbackListMap.put(familyName, fallback);
        }

        // Then, add fallback fonts to the each fallback map.
        for (int i = 0; i < xmlFamilies.length; i++) {
            final FontConfig.Family xmlFamily = xmlFamilies[i];
            // The first family (usually the sans-serif family) is always placed immediately
            // after the primary family in the fallback.
            if (i == 0 || xmlFamily.getName() == null) {
                pushFamilyToFallback(xmlFamily, fallbackListMap, bufferCache, fontDir);
            }
        }

        // Build the font map and fallback map.
        for (int i = 0; i < fallbackListMap.size(); i++) {
            final String fallbackName = fallbackListMap.keyAt(i);
            final List<FontFamily> familyList = fallbackListMap.valueAt(i);
            final FontFamily[] families = familyList.toArray(new FontFamily[familyList.size()]);

            fallbackMap.put(fallbackName, families);
            final long[] ptrArray = new long[families.length];
            for (int j = 0; j < families.length; j++) {
                ptrArray[j] = families[j].mNativePtr;
            }
            fontMap.put(fallbackName, new Typeface(nativeCreateFromArray(
                    ptrArray, RESOLVE_BY_FONT_TABLE, RESOLVE_BY_FONT_TABLE)));
        }

        // Insert alias to font maps.
        for (final FontConfig.Alias alias : fontConfig.getAliases()) {
            Typeface base = fontMap.get(alias.getToName());
            Typeface newFace = base;
            int weight = alias.getWeight();
            if (weight != 400) {
                newFace = new Typeface(nativeCreateWeightAlias(base.native_instance, weight));
            }
            fontMap.put(alias.getName(), newFace);
        }
}
```
- 首先通过FontListParser去解析fonts.xml文件，根据文件里标签数据构建数据结构。

fonts.xml 结构如下

```xml
<familyset version="23">
    <!-- first font is default -->
    <family name="sans-serif">
        <font weight="100" style="normal">Roboto-Thin.ttf</font>
        <font weight="100" style="italic">Roboto-ThinItalic.ttf</font>
        <font weight="300" style="normal">Roboto-Light.ttf</font>
        <font weight="300" style="italic">Roboto-LightItalic.ttf</font>
        <font weight="400" style="normal">Roboto-Regular.ttf</font>
        <font weight="400" style="italic">Roboto-Italic.ttf</font>
        <font weight="500" style="normal">Roboto-Medium.ttf</font>
        <font weight="500" style="italic">Roboto-MediumItalic.ttf</font>
        <font weight="900" style="normal">Roboto-Black.ttf</font>
        <font weight="900" style="italic">Roboto-BlackItalic.ttf</font>
        <font weight="700" style="normal">Roboto-Bold.ttf</font>
        <font weight="700" style="italic">Roboto-BoldItalic.ttf</font>
    </family>

    <!-- Note that aliases must come after the fonts they reference. -->
    <alias name="verdana" to="sans-serif" />
    <alias name="sans-serif-condensed-light" to="sans-serif-condensed" weight="300" />
    <alias name="sans-serif-condensed-medium" to="sans-serif-condensed" weight="500" />

    <family lang="und-Ethi">
        <font weight="400" style="normal">NotoSansEthiopic-Regular.ttf</font>
        <font weight="700" style="normal">NotoSansEthiopic-Bold.ttf</font>
        <font weight="400" style="normal" fallbackFor="serif">NotoSerifEthiopic-Regular.otf</font>
        <font weight="700" style="normal" fallbackFor="serif">NotoSerifEthiopic-Bold.otf</font>
    </family>
```
数据图：
![](res/picture/fontsData.png)
