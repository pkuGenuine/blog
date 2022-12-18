# RDF
The Resource Description Framework (RDF) is a framework for expressing information about resources.

At the lowest level, RDF are directional graphs stored with unicode triples. RDF introduce some concepts and regulations to make "" more 

RDF introduces some concepts and specifications, which makes this way of drawing relationships more standardized and general.

## Data Model

RDF allows us to make statements about resources. Each statement is just a unicode triple, which expresses a directional relationship between two resources.

~~~
<subject> <predicate> <object>
~~~

Although all three are unicode strings in essence, they have semantics and require additional formats in actual implementation.

For example, we may use `<IRI>` to represent an IRI, as discussed later while using `'some string'` to represent plain text.

### IRI

The abbreviation IRI is short for "International Resource Identifier". The URLs (Uniform Resource Locators) that people use as Web addresses are one form of IRI. 

Other forms of IRI can provide an identifier for a resource without implying its location or how to access it. For example, I may design a schema `ask:Liyue?has_charaters` which asks the brower to retrieve information about characters in Liyue from wherever it likes.

IRIs can appear in all three positions of a triple. 

### Literals
Literals are basic values that are not IRIs. Examples of literals include strings such as "La Joconde", dates such as "the 4th of July, 1990" and numbers such as "3.14159". 


A literal in an RDF graph consists of two or three elements:
- A lexical form, being a Unicode string, which should be in [Normal Form C](https://www.w3.org/TR/2014/REC-rdf11-concepts-20140225/#bib-NFC),
- A datatype IRI, being an IRI identifying a datatype that determines how the lexical form maps to a literal value.
- If and only if the datatype IRI is `http://www.w3.org/1999/02/22-rdf-syntax-ns#langString`, a non-empty language tag as defined by [BCP47](http://tools.ietf.org/html/bcp47#section-2.2.9).

Literals may only appear in the object position of a triple.

### Blank nodes
Blank nodes are disjoint from IRIs and literals. RDF makes no reference to any internal structure of blank nodes.

Blank nodes can appear in the subject and object position of a triple. They can be used to denote resources without explicitly naming them with an IRI. For example, a tree in Mona Lisa's background is not worthy to allocate an IRI.

### Hello world
TODO: Add hello world example


## RDF Vocabularies
The RDF data model provides a way to make statements about resources. However, this data model does not make any assumptions about what resource IRIs stand for. 

In practice, RDF is typically used in combination with vocabularies, or namespaces, that provide semantic information about these resources.

For example, [FOAF](http://xmlns.com/foaf/0.1/) is such a vocabulary. It provides `knows` with IRI `http://xmlns.com/foaf/0.1/knows`. With this, we can write the tuple like `<Alice> <foaf:knows> <Bob>`.

### RDF Schema
RDF Schema can be used to create hierarchies of classes and sub-classes and of properties and sub-properties. 

With `rdfs:` prefix, it provides:
- Class: `http://www.w3.org/2000/01/rdf-schema#Class`

I am not sure whether `rdf:` prefix is also part of the RDF Schema. Probably yes, and they are tightly connected.

> The fact that the constructs have two different prefixes (rdf: and rdfs:) is a somewhat annoying historical artefact, which is preserved for backward compatibility.

Prefix `rdf:` provides:
- type: `http://www.w3.org/1999/02/22-rdf-syntax-ns#type`
- Property: `http://www.w3.org/1999/02/22-rdf-syntax-ns#Property`

With `rdf:` and `rdfs`, we can declare our own class and property:

~~~
<Person> <rdf:type> <rdfs:Class>
<is a friend of> <rdf:type> <rdf:Property>
<is a friend of> <rdfs:domain> <Person>
<is a friend of> <rdfs:range> <Person>
<is a good friend of> <rdfs:subPropertyOf> <is a friend of>
~~~

BTW, `rdfs:domain` specifies that the subject can has this property. On the other hand, `rdfs:range` specifies object.

### XSD
The `xsd` namespace provides IRI for literal's type.

- date: `http://www.w3.org/2001/XMLSchema#date`
- integer: `http://www.w3.org/2001/XMLSchema#integer`
- ...


## Serialization
A number of different serialization formats exist for writing down RDF graphs.

### N-Triples

[N-TRIPLES](https://www.w3.org/TR/rdf11-primer/#bib-N-TRIPLES) provides a simple line-based, plain-text way for serializing RDF graphs.

~~~
<http://example.org/bob#me> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://xmlns.com/foaf/0.1/Person> .
<http://example.org/bob#me> <http://xmlns.com/foaf/0.1/knows> <http://example.org/alice#me> .
<http://example.org/bob#me> <http://schema.org/birthDate> "1990-07-04"^^<http://www.w3.org/2001/XMLSchema#date> .
~~~

Each line represents a triple. Full IRIs are enclosed in angle brackets `<>`. The period at the end of the line signals the end of the triple.

In line 3 we see an example of a literal, in this case a date. The datatype is appended to the literal through a `^^` delimiter. The date representation follows the conventions of the XML Schema datatype date.

### JSON-LD
TODO: Add more info

[JSON-LD](https://www.w3.org/TR/json-ld/) provides a JSON syntax for RDF graphs and datasets.

JSON-LD offers universal identifiers for JSON objects, a mechanism in which a JSON document can refer to an object described in another JSON document elsewhere on the Web, as well as datatype and language handling.

## Tools

### RDFLib

[RDFLib](https://rdflib.readthedocs.io/en/stable/index.html) is a pure Python package for working with RDF.


## References

- [RDF](https://www.w3.org/RDF/)
- [RDF Primer](https://www.w3.org/TR/rdf11-primer/#section-use-cases)
