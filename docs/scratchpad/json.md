# Canonical JSON Representation of OpenPGP Objects

This is a JSON schema adapted from the one used internally by Hockeypuck.

## Schema Definition in Go

```
type Packet struct {
    Tag    uint8  `json:"tag"`
    Data   []byte `json:"data"`
    Parsed bool   `json:"parsed"`

    // NEW FIELD required for lossless roundtripping
    Frames []*Frame  `json:"frames,omitempty"`
}

type Algorithm struct {
    Name      string `json:"name"`
    Code      int    `json:"code"`
    BitLength int    `json:"bitLength"`       // TODO: should this be optional?
    Curve     string `json:"curve,omitempty"`
}

type PublicKey struct {
    Fingerprint  string       `json:"fingerprint"`
    LongKeyID    string       `json:"longKeyID"`
    Creation     string       `json:"creation,omitempty"` // TODO: is this ever empty?
    Expiration   string       `json:"expiration,omitempty"`
    NeverExpires bool         `json:"neverExpires,omitempty"`
    Version      uint8        `json:"version"`
    Algorithm    Algorithm    `json:"algorithm"`
    Signatures   []*Signature `json:"signatures,omitempty"`
    Packet       *Packet      `json:"packet,omitempty"`

    // no longer used by hockeypuck
    ShortKeyID   string       `json:"shortKeyID"`
    BitLength    int          `json:"bitLength"` // Moved under Algorithm; not meaningful for all algorithm types
}

type PrimaryKey struct {
    *PublicKey

    MD5       string           `json:"md5"`         // SKS digest
    Length    int              `json:"length"`
    SubKeys   []*SubKey        `json:"subKeys,omitempty"`
    UserIDs   []*UserID        `json:"userIDs,omitempty"`

    // no longer used by hockeypuck
    UserAttrs []*UserAttribute `json:"userAttrs,omitempty"`
}

type SubKey struct {
    *PublicKey
}

type UserID struct {
    Keywords   string       `json:"keywords"`
    Packet     *Packet      `json:"packet,omitempty"`
    Signatures []*Signature `json:"signatures,omitempty"`
}

type Signature struct {
    // existing packet fields
    SigType      int     `json:"sigType"`
    Revocation   bool    `json:"revocation,omitempty"`
    Packet       *Packet `json:"packet,omitempty"`

    // existing subpacket-derived fields
    Primary      bool    `json:"primary,omitempty"`
    IssuerKeyID  string  `json:"issuerKeyID,omitempty"`
    Creation     string  `json:"creation,omitempty"`
    Expiration   string  `json:"expiration,omitempty"`  // EITHER sig OR key expiration
    NeverExpires bool    `json:"neverExpires,omitempty"`
    PolicyURI    string  `json:"policyURI,omitempty"`

    // NEW PACKET FIELDS
    Version         uint8        `json:"version"`
    PubkeyAlgorithm Algorithm    `json:"pubkeyAlgorithm"`
    HashAlgorithm   Algorithm    `json:"hashAlgorithm"`
    HashedArea      []*Subpacket `json:"hashedArea,omitempty"`
    UnhashedArea    []*Subpacket `json:"unhashedArea,omitempty"`

    // NEW SUBPACKET-DERIVED FIELDS
    Exportable       bool           `json:"exportable" default:"true"`
    Trust            Trust          `json:"trust,omitempty"`
    Regex            string         `json:"regexes,omitempty"`
    Revocable        bool           `json:"revocable" default:"true"`
    PrefSymmetric    []Algorithm    `json:"prefSymmetric,omitempty"`
    RevocationKeys   []string       `json:"revocationKeys,omitempty"`
    Notations        []*Notation    `json:"notations,omitempty"`
    PrefHash         []Algorithm    `json:"prefHash,omitempty"`
    PrefCompression  []Algorithm    `json:"prefCompression,omitempty"`
    KeyServerPrefs   KeyServerPrefs `json:"keyserverPrefs,omitempty"`
    PrefKeyServer    string         `json:"prefKeyserver,omitempty"`
    Flags            Flags          `json:"flags,omitempty"`
    SignerUserID     string         `json:"signerUserID,omitempty"`
    RevocationReason Reason         `json:"revocationReason,omitempty"`
    Features         Features       `json:"features,omitempty"`
    SigTarget        SigTarget      `json:"sigTarget,omitempty"`
    EmbeddedSigs     []*Signature   `json:"embeddedSigs,omitempty"` // BEWARE recursion
    Issuer           VFP            `json:"issuer,omitempty"`
    Recipients       []VFP          `json:"recipients,omitempty"`
    ApprovedCerts    [][]byte       `json:"approvedCerts,omitempty"`
    KeyBlocks        []KeyBlock     `json:"keyBlocks,omitempty"`    // BEWARE recursion
    PrefAEAD         []Algorithm    `json:"prefAEAD",omitempty"`
    PrefSuites       []AEADSuite    `json:"prefSuites,omitempty"`
    LiteralMetadata  MetaData       `json:"literalMetadata,omitempty"`
    Replacement      Replacement    `json:"replacement,omitempty"`
}

// no longer used by hockeypuck

type UserAttribute struct {
    Photos      []*Photo     `json:"photos,omitempty"`
    Packet      *Packet      `json:"packet,omitempty"`
    Signatures  []*Signature `json:"signatures,omitempty"`
    Unsupported []*Subpacket `json:"unsupported,omitempty"`
}

type Photo struct {
    MIMEType string `json:"mimeType"` // always 'image/jpeg'
    Data     []byte `json:"data"`
}

// NEW STRUCTS

type Subpacket struct {
    Tag      uint8  `json:"tag"`
    Critical bool   `json:"critical"`
    Data     []byte `json:"data"`
    Parsed   bool   `json:"parsed"` // If true, the value has been copied to the parent object
}

// packets

type SessionKey struct {
    Version   uint8     `json:"version"`
    Algorithm Algorithm `json:"algorithm"`
    Packet    *Packet   `json:"packet,omitempty"`
}

type PKESK struct {
    *SessionKey

    Recipient VFP    `json:"recipient,omitempty"`
    LongKeyID string `json:"longKeyID,omitempty"`
}

type SKESK struct {
    *SessionKey

    S2KSpecifier S2KSpecifier `json:"s2kSpecifier,omitempty"`
}

type CompressedData struct {
    Algorithm Algorithm `json:"algorithm"`
    Packet    *Packet   `json:"packet,omitempty"`
}

type LiteralData struct {    
    Metadata VerbatimMetadata `json:"metadata,omitempty"`
    Packet   *Packet          `json:"packet,omitempty"`
}

type SED struct {
    Packet    *Packet   `json:"packet,omitempty"`
}

type SEIPD struct {
    Version   uint8     `json:"version"`
    Suite     AEADSuite `json:"suite,omitempty"`     // v2
    ChunkSize uint8     `json:"chunkSize,omitempty"` // v2
    Salt      []byte    `json:"salt,omitempty"`      // v2
    Packet    *Packet   `json:"packet,omitempty"`
}

// as per draft-ietf-rfc4880bis-10
// this has had too many names, so identify by packet tag
type Type20 {
    Version   uint8     `json:"version"`
    Suite     AEADSuite `json:"suite"`
    ChunkSize uint8     `json:"chunkSize"`
    IV        []byte    `json:"iv"`
    Packet    *Packet   `json:"packet,omitempty"`
}

// packet sequences

type Message struct {
    SessionKeys    []*SessionKey   `json:"sessionKeys,omitempty"`
    CompressedData *CompressedData `json:"compressedData,omitempty"`
    LiteralData    *LiteralData    `json:"literalData,omitempty"`
    SED            *SED            `json:"sed,omitempty"`
    SEIPD          *SEIPD          `json:"seipd,omitempty"`
    Type20         *Type20         `json:"type20,omitempty"`
    Signatures     []*Signature    `json:"signatures,omitempty"`
}

// as per draft-gallagher-openpgp-hkp
// TODO: needs a better name!
type MixedKeyring struct {
    Signatures   []*Signature  `json:"signatures,omitempty"`
    Certificates []*PrimaryKey `json:"certificates,omitempty"`
}

// data types

type Frame struct {
    Length int  `json:"length"`
    Legacy bool `json:"legacy" default:"false"`
}

type VFP struct {
    Version     uint8  `json:"version"`
    Fingerprint string `json:"fingerprint"`
}

type S2KSpecifier struct {
    Type          uint8     `json:"type"`
    HashAlgorithm Algorithm `json:"algorithm,omitempty"`
    Salt          []byte    `json:"salt,omitempty"`
    Count         uint32    `json:"count,omitempty"`    // if type==3, decoded!
    Parallel      uint8     `json:"parallel,omitempty"`
    Memory        uint32    `json:"memory,omitempty"`   // decoded!
}

type AEADSuite struct {
    SymmetricAlgorithm Algorithm `json:"symmetricAlgorithm"`
    AEADAlgorithm      Algorithm `json:"aeadAlgorithm"`
}

type Trust struct {
    Depth  uint8 `json:"depth"`
    Amount uint8 `json:"amount"`
}

type SigTarget struct {
    PubkeyAlgorithm Algorithm `json:"pubkeyAlgorithm"`
    HashAlgorithm   Algorithm `json:"hashAlgorithm"`
    Digest          []byte    `json:"digest"`
}

type Reason struct {
    Code   uint8  `json:"code"`
    Reason string `json:"reason,omitempty"`
}

type Notation struct {
    Class NotationClass `json:"class"`
    Name  string        `json:"name"`
    Value string        `json:"value,omitempty"` // if human-readable
    Data  []byte        `json:"data,omitempty"`  // if not human-readable
}

// as per draft-gallagher-openpgp-literal-data-metadata
type Metadata struct {
    Encoding uint8            `json:"encoding"`
    Digest   []byte           `json:"digest,omitempty"`
    Verbatim VerbatimMetadata `json:"verbatim,omitempty"`
}

type VerbatimMetadata struct {
    Format   byte   `json:"format"`
    Filename string `json:"filename"`
    Date     string `json:"date"`
}

// as per draft-ietf-openpgp-replacementkey
type Replacement struct {
    Class   ReplacementClass `json:"class"`
    Targets []*TargetRecord  `json:"targets"`
}
type TargetRecord struct {
    Target  VFP    `json:"target"`
    Imprint []byte `json:"imprint"`
}

// as per draft-ietf-rfc4880bis-10
type KeyBlock struct {
    Class uint8       `json:"class"` // unnamed in rfc4880bis
    Cert  *PrimaryKey `json:"cert"`
}

// flag bits

type KeyserverPrefs struct {
    NoModify bool `json:"noModify" default:"false"`
}

type Flags struct {
    SigCertify   bool `json:"sigCertify" default:"false"`
    SigLiteral   bool `json:"sigLiteral" default:"false"`
    EncComms     bool `json:"encComms" default:"false"`
    EncStore     bool `json:"encStore" default:"false"`
    Split        bool `json:"split" default:"false"`
    SigAuth      bool `json:"sigAuth" default:"false"`
    Communal     bool `json:"communal" default:"false"`
    EncADSK      bool `json:"encAdsk" default:"false"` // TODO: encRestricted ?
    SigTimestamp bool `json:"sigTimestamp" default:"false"`
}

type Features struct {
    SEIPDv1  bool `json:"seipdv1" default:"false"`
    Type20   bool `json:"type20" default:"false"`
    PubkeyV5 bool `json:"pubkeyv5" default:"false"`
    SEIPDv2  bool `json:"seipdv2" default:"false"`
}

type NotationClass struct {
    HumanReadable bool `json:"humanReadable" default:"false"`
}

type ReplacementClass struct {
    Inverse bool `json:"inverse" default:"false"`
}
```

## Notes for Implementers

Applications have significant flexibility in choosing how deeply to parse OpenPGP structures, and which JSON fields to populate.
For example:

* An application MAY omit all `Packet` and/or `Subpacket` objects for brevity, or it MAY omit just those that it has parsed.

* An application MAY choose not to parse subpackets, and leave empty the corresponding fields in the parent `Signature` and/or `Packet` objects.

* If an application includes `Packet` objects it MAY omit `Frames`.

* An application MAY choose not to populate the human-readable `Name` fields in data types such as `Algorithm`.

The schema does however impose some restrictions:

* When parsing subpackets, semantic ambiguities MUST be resolved by the application before populating subpacket-derived fields, for example the application MUST decide:

    * which of the Key Expiration or Signature Expiration timestamps is applicable;
    * how to resolve multiple subpackets of the same type;
    * whether to disregard subpackets in the unhashed area.

* Algorithm-specific data in key material and signature packets cannot be represented in parsed form.

* Representation of the decompressed or decrypted contents of a Compressed Data or Encrypted Data packet is beyond the scope of this schema.

## Design Rationale

* We follow the field naming convention established in hockeypuck where possible.

* Unnamed fields have been given descriptive names.

* Where several possible names exist for a field, one has been chosen arbitrarily.

* Packets and subpackets are logically divided into structured fields where this seems useful; this choice is to some extent arbitrary.

* Where an object has only one algorithm-like field, and the algorithm registry is therefore obvious, the field is just named "algorithm".
