title: ctags tag 文件错误问题
date: 2015-01-19 10:16:16
tags: [linux]
---

有些时候用 ctags 打出来的 tag，在 vim 里跳转用的时候，有些会报什么 tag 文件格式错误，然后跳转无效的问题。我遇到好几次了，公司的电脑，自己的电脑。今天突发奇想，打开 tag 文件看了一下，发现文件开头有这么一段：

<pre config="brush:bash;toolbar:false;">
                                int  hash_idx,
                                int  iteration_count,  
                      unsigned char *out,
                      unsigned long  password_len, 
                      unsigned long *outlen)
                const unsigned char *salt, 
!_TAG_FILE_FORMAT   2   /extended format; --format=1 will not append ;" to lines/
!_TAG_FILE_SORTED   1   /0=unsorted, 1=sorted, 2=foldcase/
!_TAG_PROGRAM_AUTHOR    Darren Hiebert  /dhiebert@users.sourceforge.net/
!_TAG_PROGRAM_NAME  Exuberant Ctags //
!_TAG_PROGRAM_URL   http://ctags.sourceforge.net    /official site/
!_TAG_PROGRAM_VERSION   5.9~svn20110310 //
"   external/chromium_org/third_party/WebKit/PerformanceTests/SunSpider/tests/sunspider-0.9.1/string-tagcloud.js    /^            '\\r': '\\\\r',$/;"   p
"   external/chromium_org/third_party/WebKit/PerformanceTests/SunSpider/tests/sunspider-0.9/string-tagcloud.js  /^            '\\r': '\\\\r',$/;"   p
"   external/chromium_org/third_party/WebKit/PerformanceTests/SunSpider/tests/sunspider-1.0/string-tagcloud.js  /^            '\\r': '\\\\r',$/;"   p
#1  external/dropbear/libtomcrypt/crypt.tex /^  {                   % THESE headers.$/;"    s
#::B    external/chromium_org/third_party/JSON/JSON-2.59/blib/lib/JSON/backportPP/Compat5005.pm /^    sub B::SVp_IOK () { 0x01000000; }$/;" s
#::B    external/chromium_org/third_party/JSON/JSON-2.59/blib/lib/JSON/backportPP/Compat5005.pm /^    sub B::SVp_NOK () { 0x02000000; }$/;" s
#::B    external/chromium_org/third_party/JSON/JSON-2.59/blib/lib/JSON/backportPP/Compat5005.pm /^    sub B::SVp_POK () { 0x04000000; }$/;" s
</pre>

那几段打 ！ 应该是注释（我不会 ctags 的语法，自己猜的），然后下面应该是真实的记录函数、变量位置的数据了。那么前面那几段感觉就怪怪，于是我尝试把那几段删掉，果然 vim 里面的跳转就能用了。

有些时候就要多猜，呵呵。

