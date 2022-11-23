# Chapter 4: Encoding and Evolution


## Formats for Encoding Data

When you want to write data to a file or send it over the network, you need to encode it in a 
self-contained sequence of bytes. This process encompasses two steps:

1. The *writer* is responsible for translating data from an in-memory representation into a 
   self-contained byte sequence. This process is known as *serialization* (also known as 
   *encoding* or *marshalling*).
2. The *reader* is responsible for translating data from the byte sequence to a valid in-memory 
   representation. This process is known as *deserialization* (also known as *decoding*, 
   *unmarshalling*, *parsing*).

In order for this process to succeed, the writer and reader need to agree upon an encoding 
scheme. There are several options to consider:

- Language-specific formats
- Textual formats, such as JSON, XML, and CSV
- Binary formats, such as Protobuf and Avro

When evaluating serialization formats, we want to pay attention to:

- *Forward compatibility*: Whether newer code can read data that was written by older code
- *Backward compatibility*: Whether older code can read data that was written by newer code
- *Compactness*: How small the encoded data payload is


### Language-specific formats

Pros:

- Convenient and straightforward to implement in application code, due to native language 
  support out of the box.

Cons:

- Data that is serialized in a language-specific format is difficult to deserialize in another 
  language, and so limits the flexibility of an application's tech stack.
- In order to deserialize data into the same object types, the decoder needs to be able to 
  instantiate arbitrary classes. This poses a security threat in case an attacker is able to 
  inject byte sequences of their arbitrary classes into the application decoder, since 
  it means that they can effectively execute arbitrary code in your application.
- Language-specific formats tend to not support either forward or backward compatibility
- Language-specific formats tend to be bloated and inefficient

I would stay away from language-specific encoding and decoding for production applications.

### JSON

Pros:

- Encoded data is human-readable and can be parsed without schema knowledge.
- Well-supported across several programming languages due to its ubiquity.
- Well-supported in the browser.
- Binary encoding formats for JSON data exist and can help reduce data size.

Cons:

- Schema definition languages for JSON do exist, but they are complicated to implement and not 
  encountered often (several JSON-based document databases rely on a *schema-on-read* pattern, 
  where the reader is responsible for interpreting arbitrary data produced by the writer and 
  returned from the database). If missing, there is no write-time schema enforcement. 
  Application code is responsible for determining which schema version a particular JSON records 
  belongs to and parsing appropriately to maintain forward / backward compatibility.
- Encoded data tends to be bloated, since each record needs to contain field names along with 
  field data (since JSON tooling typically does not take advantage of a pre-defined schema). 
  Binary encoding of JSON data helps somewhat but fundamentally does not eliminate this drawback,
  as the field names still need to be encoded. 

### XML

Same as JSON

### CSV

Similar to JSON, except that CSV has no schema definition or binary encoding. In addition, CSV 
has to deal with escape character complications in case the delimiter is included in the data 
itself.

### Protobuf

Data schemas are explicitly defined in the reader and writer, and each schema is assigned a tag 
number as a unique identifier. At serialization time, data is converted to binary code and each 
field's byte string is labeled with its corresponding tag number (this is where the primary 
space savings come from compared to binary-encoded JSON - field names are swapped with more 
compact tag numbers). At deserialization time, the reader uses its copy of the Protobuf schema 
to match data by field tag.

In order to maintain forward and backward compatibility with Protobuf:

- Do not reassign a tag number to a different field, as this invalidates all existing serialized 
  data
- Do not remove a required field when updating schemas, as this invalidates any old code reading 
  new data with the omitted field (i.e., it is not backward compatible) 
- Do not introduce a required field when updating schemas, as this invalidates any new code 
  reading old data with the omitted field (i.e., it is not forward compatible)

Protobuf definitions are defined and distributed across readers and writers via source control 
(e.g. having a dedicated repository of `.proto` files for readers and writers to pull from / 
update).

Pros:

- Supports code generation at compile-time for building and manipulating Protobuf messages as 
  objects 
- Highly compact since field names are swapped with field tags

Cons:

- Schema updates need to follow very particular rules in order to avoid invalidating data or 
  compromising forward / backward compatibility
- Cannot decode data at read time without knowing the schema
- Not human-readable

### Avro

Data schemas are explicitly defined in the reader and writer, and fields are uniquely identified 
by name. At serialization time, the writer uses its schema to encode the data in binary format 
with no reference to any field (leading to an even more compact representation than Protobuf, 
since Protobuf includes field tags). At deserialization time, the reader *compares* its schema 
to the writer schema and matches each field by name to figure out how to read the written data.

This means that a reader must know the schema that a writer used in order to interpret the data. 
The reader can obtain writer schema in one of the following ways:

1. When consuming a batch of Avro files, the first file can simply contain the 
   writer Avro schema as JSON, which can be produced at write time.
2. When reading records from a database, the record can include a version number indicating 
   which version of the writer schema was used to serialize the data. The reader can refer to 
   some dedicated schema registry (e.g. another table in the database) and look up the version 
   in the record to identify the corresponding writer schema that was used.
3. When receiving records from a network connection, the reader and writer can negotiate the 
   schema at connection setup time and use it for the lifetime of the connection.

In order to maintain forward and backward compatibility with Avro:

- Do not remove a field that is not nullable, as this invalidates any old code reading new data 
  (i.e., it is not backward compatible)
- Do not add a new field that is not nullable, as this invalidates any new code reading old data 
  (i.e., it is not forward compatible)
- Note that renaming a nullable field should not break backward compatibility per se, but old code 
  reading new data will always mark the field with the original name as null and ignore the 
  field in the new name

Pros:

- Easy to dynamically generate JSON schema files (unlike in, say, Protobuf, where there is a 
  risk of accidentally reusing tag numbers)
- Highly compact since there is no schema metadata in the serialized data (even more so than 
  Protobuf due to field tags)

Cons:
 
- Cannot decode data at read time without knowing both the reader and writer schema (unlike 
  Protobuf, where the reader schema matches written data by field tag since field tags are 
  included in the serialized data)
- Not human-readable

 