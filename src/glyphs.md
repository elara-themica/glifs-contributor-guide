# The Globally Linked Indexing File System (GLIFS)

This is the GLIFS developer and contributor guide. In general, the idea of the
project is to create a kind of hybrid between IPFS and a document database:

- Closer to decentralized (like the web) than completely distributed using
  technologies like DHTs.  (They may, however, be used for some ancillary
  system-wide metadata or "storage of last resort").

- Optimized for large numbers of small, semi-structured data objects rather than
  POSIX-style unstructured (at least, from the perspective of the FS) files.

- Using Merkle DAGs and cryptographic signatures to verify collection histories.

Please be aware that development is still in the proof of concept phase.
Specifically, this means:

- While I have a good feeling for the overall goals of the project, many of the
  concepts still need to be fleshed out, and others have not been documented.
  That includes this contributor guide!

- While any potential collaborators would be given a __WARM__ welcome, for now
  the best form of that would be to reach out via email to get in touch and have
  a conversation.

# The Glyph Format

GLIFS uses a flexible semi-structured binary data storage format, with
individual items called _glyphs_. The format is influenced by systems like
[Protocol Buffers](https://en.wikipedia.org/wiki/Protocol_Buffers) and
[Cap'n Proto](https://capnproto.org/language.html), in that it is a binary
format intended to be used in-memory _directly_, without requiring any
deserialization or memory allocation on modern hardware. This allows it to be
extremely fast, a useful property when building indexes of an extremely large
number of individual objects. However, the glyphs format is _self-describing_.
Structure and type information is encoded with each object (or, at least
contains an explicit, machine-readable reference thereto), so the meaning of
glyphs can be understood without reference to any external schema information.

Current documentation on the binary format can be found in the chapter on
[binary formats](binary_formats.md).
