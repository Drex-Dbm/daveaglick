﻿Title: Strongly Typed Icon Fonts in ASP.NET MVC
Published: 1/2/2014
Tags:
  - ASP.NET
  - ASP.NET MVC
  - HtmlHelper
  - CSS
  - icon fonts
  - icons
---

<p>This a technique for working with icon fonts, which have been steadily gaining in popularity. I love icon fonts. They allow me to package up a whole bunch of simple glyphs and pictograms, use them on my site or application without too much fuss on nearly every browser, and let me control presentation attributes such as color, size, etc. I especially like the recent trend of web-based tools for building custom icon fonts from an available library of glyphs (I tend to use <a href="http://fontastic.me">Fontastic</a>, but I've also had good luck with <a href="http://icomoon.io">IcoMoon</a> and <a href="http://flaticon.com">FlatIcon</a>).</p>

<p>Most icon fonts, including those from the services mentioned above, provide a CSS file with styles you can use to insert specific icons from the font using a friendly name. Most of these style sheets either use an HTML5 <code>data-</code> declaration and/or a CSS class with a <code>:before</code> selector to insert the requested character in the appropriate font face before the target element. For example, the top of the CSS file I get with the font I use from Fontastic looks like:</p>

<pre class="prettyprint">@charset "UTF-8";

@font-face {
  font-family: "back-office";
  src:url("fonts/back-office.eot");
  src:url("fonts/back-office.eot?#iefix") format("embedded-opentype"),
    url("fonts/back-office.woff") format("woff"),
    url("fonts/back-office.ttf") format("truetype"),
    url("fonts/back-office.svg#back-office") format("svg");
  font-weight: normal;
  font-style: normal;
}

[data-icon]:before {
  font-family: "back-office" !important;
  content: attr(data-icon);
  font-style: normal !important;
  font-weight: normal !important;
  font-variant: normal !important;
  text-transform: none !important;
  speak: none;
  line-height: 1;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

[class^="icon-"]:before,
[class*=" icon-"]:before {
  font-family: "back-office" !important;
  font-style: normal !important;
  font-weight: normal !important;
  font-variant: normal !important;
  text-transform: none !important;
  speak: none;
  line-height: 1;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.icon-address-book:before {
  content: "a";
}
.icon-alert:before {
  content: "b";
}</pre>

<p>Ignore the <code>@font-face</code> declaration and the two blocks after it (the first sets up common styles when using a <code>data-icon</code> attribute and the second sets up common styles for <code>icon-</code> classes). The important thing is the last part of the file that declares the CSS classes <code>icon-address-book</code> and <code>icon-alert</code>. You typically use these in your HTML (or Razor) as:</p>

<pre class="prettyprint">&lt;span&gt;
    &lt;i class="icon-alert"&gt;&lt;/i&gt; Oh, no! There's an alert icon preceding this text!
&lt;/span&gt;</pre>

<p>This requires that you use the name of the CSS class directly in your view. That is a "<a href="http://en.wikipedia.org/wiki/Magic_number_(programming)">magic string</a>", and I <strong>hate</strong> magic strings. What if you refactor the icon font to remove a particular icon (or worse yet, one of your team members commits a new icon font without particular icons – it's happened to me)? What if you want to automatically find all of the uses of a particular icon? What if you want to pass around or store an icon without resorting to strings? All of these can be solved by using something other than a string literal to represent a particular icon.</p>

<p>The first step is figuring out what we're going to use instead. I like enums for this purpose because they're strongly-typed and not interchangeable. But there's one small problem: if I want to refer to my icons with a friendly name, I still have to store the CSS class name somewhere. Unfortunately in C# enums can't have string values. A workaround is to store the CSS class names as a <code><a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.descriptionattribute(v=vs.110).aspx">System.ComponentModel.DescriptionAttribute</a></code>. Then we can just use the following extension method to get the value of the <code>DescriptionAttribute</code> whenever we need it (just put in any static class):</p>

<pre class="prettyprint">public static string GetDescription(this Enum value)
{
    FieldInfo fieldInfo = value.GetType().GetField(value.ToString());
    DescriptionAttribute description = fieldInfo.GetCustomAttribute(false);
    return description == null ? value.ToString() : description.Description;
}</pre>

<p>I use this technique a lot outside this particular font icon problem and it works very well for attaching string values to an enum member. This lets us create an enum type that looks like this:</p>

<pre class="prettyprint">public enum Icon
{
    [Description("icon-address-book")]
    AddressBook,
    [Description("icon-alert")]
    Alert
}</pre>

<p>Then, if we create a couple of <a href="http://www.asp.net/mvc/tutorials/older-versions/views/creating-custom-html-helpers-cs">HTML helpers</a> (these can just go in any static class):</p>

<pre class="prettyprint">public static MvcHtmlString Icon(this HtmlHelper helper, Icon icon)
{
    return MvcHtmlString.Create(icon.Html());
}

// Not really an HTML helper, but included here anyway
public static string Html(this Icon icon)
{
    if (icon == Util.Icon.None) return string.Empty;
    return string.Format("&lt;i class='{0}'&gt;&lt;/i&gt; ", icon.GetDescription());
}</pre>

<p>We can write the following in our view (this assumes Razor, but the technique should work in any view engine that exposes the <code><a href="http://msdn.microsoft.com/en-us/library/system.web.mvc.htmlhelper(v=vs.118).aspx">HtmlHelper</a></code> class and supports extension methods):</p>

<pre class="prettyprint">&lt;span&gt;
    @Html.Icon(Icon.Alert) Oh, no! There's an alert icon preceding this text!
&lt;/span&gt;</pre>

<p>This successfully eliminated our magic string. But we still have to create the <code>Icon</code> enum and manually match it to the CSS from the icon font. Luckily, there is an easy way to create the enum automatically using T4 templates. Generally I like to limit the number of T4 templates I have in my projects. They eat up time in the build cycle and can get out of sync with the current build causing some hard to diagnose and often frustrating bugs. But in cases like this, T4 is perfect. Just drop the following into your project and name it "Icons.tt".</p>

<pre class="prettyprint">&lt;#@ template language="C#" hostSpecific="true" #&gt;
&lt;#@ assembly name="System.Core" #&gt; 
&lt;#@ import namespace="System.Linq" #&gt;
&lt;#@ import namespace="System.Collections.Generic" #&gt;
&lt;#@ import namespace="System.Text.RegularExpressions" #&gt;
&lt;# Process(); #&gt;
&lt;#+
    readonly Regex regex = new Regex(@"^\.icon-(.*)\:before \{$", RegexOptions.Compiled | RegexOptions.Multiline);

    public void Process()
    {
        WriteLine("using System.ComponentModel;");
        WriteLine("");
        WriteLine("namespace Sipc.BackOffice.Util");
        WriteLine("{");
        WriteLine("\tpublic enum Icon");
        WriteLine("\t{");
        string css = System.IO.File.ReadAllText(Host.ResolvePath("..\\Content\\icons\\styles.css"));
        foreach (Match match in regex.Matches(css))
        {
            WriteLine("\t\t[Description(\"icon-" + match.Groups[1].Value + "\")]");
            WriteLine("\t\t" + String.Join(null, match.Groups[1].Value.Split(new char[]{'-'}, StringSplitOptions.RemoveEmptyEntries)
                .Select(x =&gt; (char.IsDigit(x[0]) ? ("_" + x[0]) : char.ToUpper(x[0]).ToString()) + x.Substring(1))) + ",");            
        }
        WriteLine("\t\t[Description(\"\")]");
        WriteLine("\t\tNone");
        WriteLine("\t}");
        WriteLine("}");
    }
#&gt;</pre>

<p>You'll need to adjust the path to the CSS file so it matches your project. You may also need to tweak the regular expression and/or the string processing (which basically just tries to strip the <code>icon-</code> part of the CSS class, remove the hyphens, and title case the rest). However, this should give you enough to start with. Once it's in the project and you rebuild, you'll get a C# source file with an <code>Icon</code> enum that has values and corresponding <code>DescriptionAttributes</code> for every class in the CSS. Then when you go to update or change your icon font, just drop the new CSS file on top of the old one and rerun the T4 template (or rebuild the project).</p>