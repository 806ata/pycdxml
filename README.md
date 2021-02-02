## ChemDraw Converter

ChemDraw converter converts between cdx (binary) and cdxml(text/xml) files containing small molecules. The goal of the project is to provide conversion for files containing small molecules and later possibly reactions, at least the reaction scheme. The idea is to be able to convert such files coming or going into a database automatically on any OS and hence treating them as molecule file format. 

Biologics are out of scope especially because these are rather new and missing from specification. Out-of-scope are also many other drawing-related things one can to in ChemDraw. The core issue is that ChemDraw is essentially a drawing canvas and hence cdx and cdxml are drawing formats and not chemical structure exchange formats.

Note that there is no chemical knowledge in this tool! It really is just a format converter.

### ChemDraw Format Specification

#### Issues

The [official specification](https://www.cambridgesoft.com/services/documentation/sdk/chemdraw/cdx/General.htm) is very much outdated. It's contradicts itself in some places any many properties and objects are outright missing or seem to have changed their type. Some of this issues have been solved simply by changing the type or reverse-engineering the property if it is simple enough.

There is an [updated header file](http://forums.cambridgesoft.com/messageview.aspx?catid=12&threadid=3822) on old CambridgeSoft forums but it doesn't explain the type or usage of new object or properties so it's of limited value.

A much bigger issue are inconsistent data types. By creating trivial test files I found that properties `color(UNINT16) `and `BracketUsage (INT8)` can appear with wrong number of bytes in the cdx file (4 and 2). These additional bytes are 0 and it's unclear what they are for. Removing the 0-bytes from color leads to a file that looks exactly the same in ChemDraw UI. The code now keeps these bytes in mind and writes them out again to create a 100% identical cdx on binary level. On cdxml these additional bytes do not appear. Hence their reason to exist is unclear.

#### CDX vs CDXML

cdx format internal is a tree-structure just like xml is. However some objects are implemented differently. For example font- and colortables are properties in cdx file they are separate objects in cdxml. For text the text style is a property in cdx but in cdxml each style is a separate object. The point being that there is no simple 1:1 mapping possible. This leads to ugly "hacks" and conditionals needed at the right places.

### Architecture / Code

Let's just say it's a big mess and there is a lot of room for improvement. I started the project by being able to import cdx and then write it out as 100% equal on byte level again. Hence the internal representation is based on the cdx specifications and not the cdxml one. There for when reading cdxml, a conversion to internal format must happen and same when writing cdxml.

#### High-level

The high-level architecture is the same as in the cdx specification. A file consists of objects. Objects have properties (or attributes in cdxml) and every attribute is of a certain type. A type can be complex or just a standard numeric type like int16. 

Objects -> Properties -> Types

The internal tree-structure is built using `anytree` library. 

#### Reading / Writing

Ignoring some exceptions, the types know how to read and write themselves. In cdx each object has it's tag id of 2 bytes. `cdx_objects.yml` maps the tag to the object name and element tag in cdxml. The object then reads all it's properties. Like objects, properties have a tag id of 2 bytes. `cdx_properties.yml` maps this tag id to the properties name and it's type. So if new objects and properties are found in the specification or by reverse-engineering,  they can be added to these files.

The property then determines it's type and the type object then reads in the data from the file or when writing generates the output data either by appending to a byte stream or by adding attributes to an lxml element object. 

The issues with this basic read/write mechanism is, that the reading and writing can all work without error but the resulting file might still not be readable by ChemDraw either due to a error in the code or an unknown object or property. Currently unknown elements simply get ignored (and logged).