# tarfs
> Backwards-compatible extension for tar archives

## Introduction
This repository describes a backwards-compatible extension to the tar file format,
which can be used to speedup reading specific members from a tar file,
especially if the file is read from a linear medium such as tape as
long as the medium is seekable.

## Details
``tarfs`` provides an index for a given tar archive which holds information
about a its members or a subset of its members.
This information includes the offset of the member in the tar archive and
thus can be used to directly seek to the position of an indexed member
instead of the need to process all preceding members of the archive.

For new tar archives this index can be included *at the start*
of the archive, or the index can be added later by replacing member(s) at the
start of the archive with the index while appending the replaced members
at the end of the archive.
If the tar archive cannot or should not be modified the index can still
be used by providing it via a file external to the archive.

A ``tarfs`` archive is simply a tar archive,
in which the first member has the name ``.tarfs`` and its data is a valid
index for the remaining archive.
Programs supporting the ``tarfs``-extension can use the index in order
to list and directly extract all members which are listed in the index.
Programs which do not support the ``tarfs``-extension simply extract the
index as a regular file called ``.tarfs``.
Using this technique a ``tarfs``-file present on a tape can be mounted like a
filesystem by first saving the index into memory or unto another filesystem.

## File format

### Internal and external indexes
A ``tarfs`` index holds exactly the same data whether it's part of an
``tarfs`` archive or provided externally.
Concatenating a tar archive with an ``tarfs`` index as its only member
and a the tar archive, which is described by the index **must** result
in a ``tarfs`` archive with a valid index.
Given that ``.tarfs`` is an index for the archive ``files.tar``,
the following commands **must** result in a valid ``tarfs`` named ``archive.tar`` - independent of the ``tarfs`` version:

```bash
$ ls -A
files.tar  .tarfs
$ tar -cf archive.tar .tarfs
$ tar -Af archive.tar files.tar
```

### Placement of the internal index
The main intent of the ``tarfs`` extension is to provide additional
functionality for already existing technologies and data.
Because the index member merely improves performance when handling the archive,
and no data is lost, when an index cannot be found,
some practical considerations concerning the meaning of "start of the archive"
seem in order.

A program implementing the ``tarfs`` extension may therefore:
 1. Provide an option to define, how many members at the start of the
    archive should be inspected when searching for the index.
    This value **should** default to ``1``.
 2. If no such option is given, the program **should** only inspect the
    first member.
 3. If the program is aware of the ``ustar``-extension for tar files,
    it **may** skip special files (with a type flag, which doesn't include
    ``NUL, '0'-'7'``) while searching for the "first" member.
 4. If the program is aware of the ``ustar``-extension it **may** mark
    the index file itself with a special type flag, so that other programs
    may exclude it when extracting files.

One example which lead to there considerations are the GNU tar extension
for [archive labels](https://www.gnu.org/software/tar/manual/html_node/label.html#label).
These labels must be present in the first member for the ``--test-label``
option to work and since replacing the label with an index seems like a very bad
idea, some flexibility seems in order.

> Note: Before implementing 4. it might be helpful to find some consensus
> how the index should be flagged.

### Versioning
As a general rule the version of an ``tarfs``-Archive is derived from
the version of the contained index.
That means no extra information is present in the ``.tarfs`` member header
in the archive for a given version *V* a ``tarfs`` archive can
be created by simply creating a tar archive and providing an index file of Version *V* as a first member.

Version information is provided by a major and a minor version number.
A program which supports a ``tarfs`` index of version ``2.1`` must
be able to work with indexes *any* version ``2.*`` (e.g. ``2.0`` and ``2.2``),
but operation may be less efficient.

### Index layout

<div align="center">

| Field offset | Field size | Field                               |
|:------------:|:----------:|:-----------------------------------:|
| 0            |   10       | ``.tar-index``                      |
| 11           |   14       | version                             |
| 26           | ...        | index data<br/> (version dependent) |
</div>

### Index v1.0
An index v1.0 file that can be used to access *n* members of a tar
archive has the size ``(n + 1) * 512`` Byte.

The first 512 Bytes contain meta-data for the index file such as the
index version and space for future features.

The remaining blocks contain a modified tar member header
of each indexed member, which contains all information of the original
tar member header as well as the position of the member in the archive.

If the index is external to the archive the position describes the offset
in the tar archive (within the compressed stream, in case the file is compressed).
If the index is part of a ``tarfs``-archive the position describes the offset
in the ``tarfs``-archive *after* the index.
Thus a ``tarfs`` index contains the same data independent of whether it's
part of the ``tarfs`` archive or provided as an external file (as stated above).

The ``tarfs`` index version ``v1.0`` has the following layout:
<div align="center">

| Field offset | Field size | Field                |
|:------------:|:----------:|:--------------------:|
| 0            |   10       | ``.tar-index``       |
| 11           |   14       | ``v1.0`` + 10 spaces |
| 26           |   488      | reserved             |
| 512          |   512      | member 1 info        |
| 1024         |   512      | member 2 info        |
| ...          | ...        | ...                  |
</div>

#### Member Info Block
The member info block has the same format as a tar
[member header](https://en.wikipedia.org/wiki/Tar_(computing)#Header),
*except* for the checksum (the 8 bytes 148-155),
which is replaced by a field holding the checksum *and* the member offset.
Because the checksum is saved as octal in tar member headers we can store
additional information when saving it as binary:

<div align="center">

| Field offset | Field size | Field      | Format        |
|:------------:|:----------:|:----------:|:-------------:|
| 148          |     6      | Checksum   | Octal (ASCII) |
| 154          |     1      | NUL        |               | 
| 155          |     1      | Space      |               |

(original)
</div>

Since ``8^6 = 262144 = 2^18`` the checksum can be represented by 18 Bits
in binary (which is rounded up to 3 Bytes), resulting in the following
adjusted fields:

<div align="center">

| Field offset | Field size | Field            | Format                       |
|:------------:|:----------:|:----------------:|:----------------------------:|
| 148          |     5      | Member position  | Unsigned integer, big endian |
| 153          |     3      | Checksum         | Unsigned integer, big endian |

</div>

> Note: Technically the checksum could be fully removed from the index,
> since it's also present in the indexed tar archive,
> but the checksum itself can be used in order to check whether the index
> itself holds faulty meta-data.
> Additionally already existing parsing code for tar member headers can
> be used to parse the index data, so a program supported ``v1.*`` indexes
> can be easily written.
> Nevertheless there's a lot of room for optimizations.

#### Member position
Because tar archive members are aligned to 512 Byte blocks,
the position of a member can be represented as the block number
(after the index).
With 5 Bytes available for the member position we can address
``2^(5*8) = 1099511627776`` blocks thus covering tar archives up to 512TB
data (after the index).
The member position points to the member *header* not the member *data*.
So the position for the member directly following the index is ``0``
(it would be ``1``, if we point to the member data).
