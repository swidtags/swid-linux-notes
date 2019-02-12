
# SWID Linux implementation notes

This repository aims to document implementation possibilities and
decisions for SWID tags on Linux.

## Standard and guidelines

### Standard version

When implementing SWID on Linux, only the ISO/IEC 19770-2:2015 standard
is considered.

* **STANDARD-1**: Only ISO/IEC 19770-2:2015 is supported.

The 2009 version of the standard is not supported.

### XML schema document

The XML namespace to be used for SWID tag XML documents is
`http://standards.iso.org/iso/19770/-2/2015/schema.xsd`.
However, that URL contains XSD for XML namespace
`http://standards.iso.org/iso/19770/-2/2014-DIS/schema.xsd` and thus
is not usable directly.

The location of the XML schema definition is
http://standards.iso.org/iso/19770/-2/2015-current/schema.xsd.
Retrieving that URL via HTTP leads to immediate `302 Found` HTTP
redirect to
https://standards.iso.org/iso/19770/-2/2015-current/schema.xsd
(with `https://`).
To assist parsers that might want to do schema validation,
`xsi:schemaLocation` may be used in SWID tags.

* **XML-1**: Use `xsi:schemaLocation` in XML documents that use
`xmlns="http://standards.iso.org/iso/19770/-2/2015/schema.xsd"`, to
steer parsers to the correct `2015-current/schema.xsd` location.

The actual XML snippet can be either
```
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://standards.iso.org/iso/19770/-2/2015/schema.xsd http://standards.iso.org/iso/19770/-2/2015-current/schema.xsd"
```
or
```
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://standards.iso.org/iso/19770/-2/2015/schema.xsd https://standards.iso.org/iso/19770/-2/2015-current/schema.xsd"
```

The first form matches the XSD URL from the section 6.1.1 of the
standard, the second form with `https://` will help parsers that do not
handle the HTTP redirect (for example `xerces-j2-2.11.0-34.fc29.noarch`
or `xjparse-1.0-18.fc29.noarch`).

### Additional guidelines

The standard primarily describes the goals and schema of the XML SWID
tag files but it does not get into great detail about practial
implementation. For that, [Practial Guidelines for the Creation
of Interoperable Software Identification (SWID) Tags:
National Institute of Standards and Technology (NIST) Internal
Report (NIST IR) 8060](https://csrc.nist.gov/publications/detail/nistir/8060/final)
is a good resource, in spite of being inconsistent at times.

* **STANDARD-2**: NIST IR 8060 should be used as a basis of
implementation decisions.

## SWID tag content

### Architecture

The standard does not provide any explicit support for holding
information about architecture for which the software is compiled.
However, the `Meta` type and element allow

```
<xs:anyAttribute processContents="lax">
  <xs:annotation>
    <xs:documentation>
      Permits any user-defined attributes in Meta tags
    </xs:documentation>
  </xs:annotation>
</xs:anyAttribute>
```

So attributes not explicitly described in the standard can be added,
for example

```
<Meta product="bash" colloquialVersion="4.4.23" revision="6.fc29" arch="x86_64"/>
```

* **TAG-1**: Use `Meta/@arch` attribute to store architecture of the software.

Note that 
[SWID Tag Validation Tool (swidval) 0.5.0](https://csrc.nist.gov/Projects/Software-Identification-SWID/resources#swid-validation-tool)
swidval will report
```
GEN-1-1: cvc-complex-type.3.2.2: Attribute 'arch' is not allowed to appear in element 'Meta'.
```
in spite of the XML schema allowing the extra attribute.

## Processing SWID tags on a host

The standard (section 6.1.4) says that SWID tags for software should
be stored in directory `swidtag` somewhere in the directory tree
where the software is installed.

### Software installed via package manager

Typical Linux installation uses package manager which installs dozens
to thousands of packages, placing files under the root (`/`)
directory. Placing their SWID tags to top level `/swidtag` directory
does not comply with Linux Filesystem Hierarchy Standard. The
`/usr/lib` is more reasonable location.

* **READ-1**: SWID tags that describe software installed via native
package manager of the operating system should be stored under
`/usr/lib/swidtag`.

### Finding all SWID tags

So to list all installed software (and SWID tags for it), one would
need to scan the whole filesystem for directories `swidtag`. That may
not be feasible for performance reasons. Also, if the host has some
remote filesystems mounted (e.g. via NFS), they may contain
directories with software which is not installed or even runnable on
the host due to missing dependencies or different architecture, and
scanning all filesystems would include those `swidtag` directories.

To make the sofware listing efficient, list of directories where
`.swidtag` files are stored can be created.

* **READ-2**: Directory `/etc/swid/swidtags.d` should contain symbolic
links to all locations where `.swidtag` files matching software
installed on the host should be looked up.

## Open questions

### Distribution of SWID XSD files

The [XSD
file](https://standards.iso.org/iso/19770/-2/2015-current/schema.xsd)
states

> ISO and IEC grant the users of this Standard the right
> to use this XSD file free of charge for the purpose of implementing
> the present Standard.

It is however not clear if distributions are allowed to distribute the
XSD file, to help users with schema validation without network access,
possibly configuring it for the users using XML catalog.

