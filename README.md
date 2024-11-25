# FOP/Saxon Issue with AreaTree

This repository reconstructs an issue with Apache-FOP and Saxon. The case is:

* We generate the FOP areatree (using `application/X-fop-intermediate-format` as output format)
* We have an element with a custom namespace in the FO input which shall be taken over to the areatree.

The problem is that the namespace declaration is not serialized so the result areatree is not wellformed.

Why this is an issue of FOP **OR** Saxon? 

**Because with Xalan it works!** The open question is: is this a problem with Saxon or does FOP use an API in a wrong way?

## The Error Details

The [input.fo](./input.fo) contains inside of `fo:declaration` a `<pdf:catalog>`:

```xml
<fo:root xmlns:fo="http://www.w3.org/1999/XSL/Format" xmlns:fox="http://xmlgraphics.apache.org/fop/extensions">
    <!-- ... -->
    <fo:declarations>
        <pdf:catalog xmlns:pdf="http://xmlgraphics.apache.org/fop/extensions/pdf">
            <!-- ... -->
        </pdf:catalog>
    </fo:declarations>

```

The area tree [result of FOP+Saxon](https://github.com/nkutsche/issue-fop-areatree-saxon/actions/runs/11915871500/job/33207176196#step:7:15) looks like:

```xml
<document xmlns="http://xmlgraphics.apache.org/fop/intermediate"
          version="2.0">
   <header>
      <pdf:catalog>
         <!-- ... -->
      </pdf:catalog>
```

The declaration of prefix `pdf` is missing, so the result is **not wellformed**!

If the areatree is generated [with FOP+Xalan the result](https://github.com/nkutsche/issue-fop-areatree-saxon/actions/runs/11915871500/job/33207176196#step:6:10) is wellformed:

```xml
<document xmlns="http://xmlgraphics.apache.org/fop/intermediate" version="2.0">
<header>
<pdf:catalog xmlns:pdf="apache:fop:extensions:pdf">
<!-- ... -->
</pdf:catalog>
```


### Overview / Stack Trace

This was my stack trace of debugging: 

```
startElement:111, XMLIndenter (net.sf.saxon.serialize)
startElement:80, NamespaceDifferencer (net.sf.saxon.event)
startElement:140, ProxyReceiver (net.sf.saxon.event)
startElement:85, SequenceNormalizer (net.sf.saxon.event)
startElement:386, ReceivingContentHandler (net.sf.saxon.event)
startElement:185, DelegatingContentHandler (org.apache.fop.util)
...

```

(So the entry point of FOP -> Saxon is [here](https://github.com/apache/xmlgraphics-fop/blob/2f1858567a8d68bd9c09eb33955d2cbae2aa03ac/fop-core/src/main/java/org/apache/fop/util/DelegatingContentHandler.java#L185))

### Saxon Entry Point

FOP calls the `net.sf.saxon.event.ReceivingContentHandler` from the `org.apache.fop.util.DelegatingContentHandler`

startElement:185, DelegatingContentHandler (org.apache.fop.util)

```java

    public void startElement(String uri, String localName, String qName,
                Attributes atts) throws SAXException {
        delegate.startElement(uri, localName, qName, atts);
    }

```

This are the parameter values:

* `uri` -> `apache:fop:extensions:pdf`
* `localName` -> `catalog`
* `qName` -> `pdf:catalog`
* `atts` is an empty list

### Saxon Intern

That is the startElement method of the `ReceivingContentHandler`: 

startElement:386, ReceivingContentHandler (net.sf.saxon.event)

```java
    public void startElement(String uri, String localname, String rawname, Attributes atts)
            throws SAXException {
        //System.err.println("ReceivingContentHandler#startElement " + localname + " sysId=" + localLocator.getSystemId());
        //for (int a=0; a<atts.getLength(); a++) {
        //     System.err.println("  Attribute " + atts.getURI(a) + "/" + atts.getLocalName(a) + "/" + atts.getQName(a));
        //}
        try {
            flush(true);

            int options = ReceiverOption.NAMESPACE_OK | ReceiverOption.ALL_NAMESPACES;
            NodeName elementName = getNodeName(uri, localname, rawname);

            AttributeMap attributes;

            if (atts.getLength() == 0) {
                attributes = EmptyAttributeMap.getInstance();
            }
            else
            {
                attributes = makeAttributeMap(atts, localLocator);
            }

            receiver.startElement(elementName, Untyped.getInstance(),
                                  attributes, currentNamespaceMap,
                                  localLocator, options);

            localLocator.levelInEntity++;
            namespaceStack.push(currentNamespaceMap);
            afterStartTag = true;

        } catch (XPathException err) {
            throw new SAXException(err.maybeWithLocation(localLocator));
        }
    }
```

It provides the `currentNamespaceMap` and creates a QName as element name and calls the receiver (`net.sf.saxon.event.SequenceNormalizer`) which calls the `net.sf.saxon.event.ProxyReceiver`.

Inbetween there are two other levels but there does not happen very much. On the third level the `net.sf.saxon.event.NamespaceDifferencer` makes a diff from the new `namespaces` to the parent namespaces (`parentMap`):

startElement:80, NamespaceDifferencer (net.sf.saxon.event)


```java

@Override
    public void startElement(NodeName elemName, SchemaType type,
                             AttributeMap attributes, NamespaceMap namespaces,
                             Location location, int properties)
            throws XPathException {
        NamespaceMap parentMap = namespaceStack.peek();
        namespaceStack.push(namespaces);
        NamespaceMap delta = getDifferences(namespaces, parentMap, elemName.hasURI(NamespaceUri.NULL));
        nextReceiver.startElement(elemName, type, attributes, delta, location, properties);

    }

```

The diff (`delta`) is empty, which is wrong I think.
