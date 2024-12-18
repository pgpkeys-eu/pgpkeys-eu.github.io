# Canonical JSON Representation of OpenPGP Objects

This is a JSON schema adapted from the one used internally by Hockeypuck.

# Generic schema

```
type Packet struct {
	Tag    uint8  `json:"tag"`
	Data   []byte `json:"data"`
	Parsed bool   `json:"parsed"`

    // NEW FIELDS required for lossless roundtripping
    Frames []*Frame  `json:"frames,omitempty"`
}

type algorithm struct {
	Name string `json:"name"`
	Code int    `json:"code"`
}

type PublicKey struct {
	Fingerprint  string       `json:"fingerprint"`
	LongKeyID    string       `json:"longKeyID"`
	Creation     string       `json:"creation,omitempty"`
	Expiration   string       `json:"expiration,omitempty"`
	NeverExpires bool         `json:"neverExpires,omitempty"`
	Version      uint8        `json:"version"`
	Algorithm    algorithm    `json:"algorithm"`
	BitLength    int          `json:"bitLength"`
	Signatures   []*Signature `json:"signatures,omitempty"`
	Packet       *Packet      `json:"packet,omitempty"`

    // no longer used by hockeypuck
	ShortKeyID   string       `json:"shortKeyID"`

    // NEW FIELDS
    Curve        curve        `json:"curve"`
}

type PrimaryKey struct {
	*PublicKey

	MD5       string           `json:"md5"` // SKS digest of the certificate
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

    // existing subpacket fields
	Primary      bool    `json:"primary,omitempty"`
	IssuerKeyID  string  `json:"issuerKeyID,omitempty"`
	Creation     string  `json:"creation,omitempty"`
	Expiration   string  `json:"expiration,omitempty"`  // this is EITHER the signature expiration OR key expiration, depending on context
	NeverExpires bool    `json:"neverExpires,omitempty"`
	PolicyURI    string  `json:"policyURI,omitempty"`

    // NEW PACKET FIELDS
	Version      uint8        `json:"version"`
	Algorithm    algorithm    `json:"algorithm"`
    HashAlgorithm algorithm   `json:"hashAlgorithm"`
    HashedArea   []*Subpacket `json:"hashedArea,omitempty"`
    UnhashedArea []*Subpacket `json:"unhashedArea,omitempty"`

    // NEW SUBPACKET FIELDS
    Exportable      bool        `json:"exportable" default:"true"`
    Trust           Trust       `json:"trust,omitempty"`
    Regexes         []string    `json:"regexes,omitempty"`
    Revocable       bool        `json:"revocable" default:"true"` // deprecated in draft-dkg-openpgp-revocation
    PrefSymmetric   []algorithm `json:"prefSymmetric,omitempty"`
    RevocationKeys  []string    `json:"revocationKeys,omitempty"`
    Notations       []*Notation `json:"notations,omitempty"`
    PrefHash        []algorithm `json:"prefHash,omitempty"`
    PrefCompression []algorithm `json:"prefCompression,omitempty"`
    KeyServerPrefs  []byte      `json:"keyserverPrefs,omitempty"` // parse bits?
    PrefKeyServer   string      `json:"prefKeyserver,omitempty"`
    Flags           []byte      `json:"flags,omitempty"`         // parse bits?
    SignerUserIDs   []string    `json:"signerUserIDs,omitempty"`
    RevocationReason Reason     `json:"revocationReason,omitempty"`
    Features        []byte      `json:"features,omitempty"`      // parse bits?
    SigTargets      []SigTarget `json:"sigTargets,omitempty"`
    Signatures      []*Signature `json:"signatures,omitempty"`   // danger of recursion!
    Issuer          VTFP        `json:"issuer,omitempty"`
    Recipients      []VTFP      `json:"recipients,omitempty"`
    ApprovedCerts   [][]byte    `json:"approvedCerts,omitempty"`
    KeyBlocks       []KeyBlock  `json:"keyBlocks,omitempty"`     // danger of recursion!
    PrefAEAD        []algorithm `json:"prefAEAD",omitempty"`     // deprecated
    PrefSuites      []AEADSuite `json:"prefSuites,omitempty"`
    LiteralMeta     Meta        `json:"literalMeta,omitempty"`
    Replacement     Replacement `json:"replacement,omitempty"`
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
	Parsed   bool   `json:"parsed"` // If true, the value has been applied to the parent object
}

type SessionKey struct {
	Version   uint8     `json:"version"`
    Algorithm algorithm `json:"algorithm"`
    Packet    *Packet   `json:"packet,omitempty"`
}

type PKESK struct {
    *SessionKey

    Recipient VTFP   `json:"recipient,omitempty"`
	LongKeyID string `json:"longKeyID,omitempty"`
}

type SKESK struct {
    *SessionKey

    S2KType uint8  `json:"s2ktype"`
}

type Message struct {
    // this doesn't look inside compressed data packets
    SessionKeys []*SessionKey `json:"sessionKeys,omitempty"`
    Contents    *Packet       `json:"contents"`
    Signatures  []*Signature  `json:"signatures,omitempty"`
}

// as per draft-gallagher-openpgp-hkp
type MixedKeyring struct {
    Signatures   []*Signature  `json:"signatures,omitempty"`
    Certificates []*PrimaryKey `json:"certificates,omitempty"`
}

type Frame struct {
    Length int  `json:"length"`
    Legacy bool `json:"legacy" default:"false"`
}


type Curve struct {
    Name string `json:"name"`
    OID  string `json:"oid"`
}

type VTFP struct {
    Version     uint8  `json:"version"`
    Fingerprint string `json:"fingerprint"`
}

type AEADSuite struct {
    Symmetric algorithm `json:"symmetric"`
    AEADMode  algorithm `json:"aeadMode"`
}

type Trust struct {
    Depth  uint8 `json:"depth"`
    Amount uint8 `json:"amount"`
}

type SigTarget struct {
    Algorithm     algorithm `json:"algorithm"`
    HashAlgorithm algorithm `json:"hashAlgorithm"`
    Digest        []byte    `json:"digest"`
}

type Reason struct {
    Code   uint8  `json:"code"`
    Reason string `json:"reason",omitempty`
}

type Notation struct {
    Class []byte `json:"class"`           // parse flags?
    Name  string `json:"name"`
    Value string `json:"value,omitempty"` // if human-readable
    Data  []byte `json:"data,omitempty"`  // if not human-readable
}

// as per draft-gallagher-openpgp-literal-data-metadata
type Meta struct {
    Class    byte   `json:"class"`
    Digest   []byte `json:"digest",omitempty`
    Date     string `json:"date",omitempty`
    Filename string `json:"filename",omitempty`
}

// as per draft-ietf-openpgp-replacementkey
type Replacement struct {
    Class   byte            `json:"class"`   // parse flags?
    Targets []*TargetRecord `json:"targets"`
}
type TargetRecord struct {
    Target  VTFP   `json:"target"`
    Imprint []byte `json:"imprint"`
}

// as per draft-ietf-rfc4880bis-10
type KeyBlock struct {
    Class   uint8       `json:"class"`
    Cert    *PrimaryKey `json:"cert"`
}
```
