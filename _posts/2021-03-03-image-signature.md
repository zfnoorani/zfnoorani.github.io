---
layout: single
classes: wide
title: "System for Verifying Authenticity of Digital Art: Prototyping"
date: 2021-03-03 12:00:00 -0600
categories: go prototypes cryptography
---

An incident on Twitter involving one of my favorite artists got me thinking about cryptographic authenticity.
One if their pieces had been edited by white supremecists to spread propaganda, and the artist was facing backlash.

*Update 3/12/2021: Note that this article doesn't [address NFT](https://www.latimes.com/business/technology/story/2021-03-11/nft-explainer-crypto-trading-collectible). This is a far less sophisticated (and far more environmentally sustainable) approach, made for learning purposes.*

I'd like to see a solution that empowers artists to sign off on copies of their digital or digitized art.
Sites that host pictures should use this to indicate to viewers whether what they're viewing is authentic or not.

Ideally, this protects artists, viewers, and the art itself.
- Artists are free from blame for appropriated art. If the image doesn't pass the authenticity check, they didn't make it!
- Those who propagate unauthorized modifications can be held accountable.
- Original copies of art, with 100% quality and the original artists' intent are marked as such.

Such a solution faces more political problems than technical.
Still, for learning, let's try conjuring something up in Go.

## Design
For this post, I'm going to address the root level problem: how to verify the authenticity of images?
This is what [digital signatures](https://en.wikipedia.org/wiki/Digital_signature) are for.

I have simple requirements:
- The image should remain readable by all existing image readers.
- The digital signature should accompany the image, without downloading a separate file.
- No modifications to the raw pixel data.

Surprisingly, there seems to be no standard or existing means of digitally signing pictures, or at least none that I've found.
Any suggested approaches I found simply sign the whole file and transmit the digital signature separately.
There's some interest, both [academic](https://people.csail.mit.edu/kimo/publications/jpeg/tifs11a.pdf) and [otherwise](https://photo.stackexchange.com/questions/15307/can-digital-cameras-sign-images-to-prove-authenticity), but mostly for images taken from digital cameras.

*Update 3/12/2021: Yes, [NFT is a thing](https://en.wikipedia.org/wiki/Crypto_art) and has been for many years. Google is hard.*

I see a few options:
1. Create a new file format with authentication in mind. 
2. Create a digital signature and append it to the file.
3. Create a digital signature and store it as [EXIF metadata](https://en.wikipedia.org/wiki/Exif).

If you've seen [XKCD #927](https://xkcd.com/927/), you'll know option one is a bad idea.

The EXIF option seems most viable to me.
Any existing application interested in checking image signatures can read EXIF metadata better than they can read garbage at the end of a JPEG.
EXIF is also a standard means of storing metadata across multiple image formats.

This idea isn't bullet proof:
- EXIF metadata is [commonly truncated](https://stackoverflow.com/questions/8774758/is-exif-data-stripped-by-ios-and-or-twitter-on-tweets-with-images). 
- There is no [EXIF field](https://exiftool.org/TagNames/EXIF.html) for digital signatures.

For now these blockers are somewhat minor.
If this idea turns out to be workable, these problems can be worked out through politics.

Let's get started.

## Choosing the Metadata

Of the EXIF fields available, I'm going to use the "Comment" field. Let's get this struct in there, encoded in JSON.

```go
type Metadata struct {
	// PublicKey is the PEM encoded public key used to sign this picture.
	PublicKey []byte
	
	// Signature is the encrypted hash of the raw pixel data.
	Signature []byte
}
```

Storing the public key here might make you cross eyed, but remember we don't have a centralized means of distributing public keys right now.

The signature is just built off of raw pixel data.
I don't think any other file metadata are important, but I'm open to changing that opinion.

There's probably smarter encodings available than PEM and JSON, with regards to message length.
Let me know if so.

## Getting a Key Pair
This scheme needs a public and private key.
For this prototype, I'm using an RSA 2048 key generated with OpenSSL.
Use these commands:
```bash
openssl genrsa -out privkey.pem
openssl rsa -in privkey.pem -pubout -out pubkey.pem
```

[See](https://www.openssl.org/docs/man1.0.2/man1/genrsa.html) `man(1) genrsa`.

## Reading and Writing EXIF Metadata
The scene for Go EXIF metadata libraries is not well developed.
There's [github.com/rwcarlsen/goexif/exif](https://github.com/rwcarlsen/goexif) and [github.com/dsoprea/goexif/v3](https://github.com/dsoprea/go-exif).
The former is feature incomplete and not being maintained
The latter pulls in too many dependencies and makes questionable use of `panic` and `defer`.

[ExifTool](https://en.wikipedia.org/wiki/ExifTool) is mature and available on most platforms.
`sudo dnf install exiftool -y` is the incantation on Fedora to install it.
It's a standalone utility written in Perl, so we'll write a wrapper around ExifTool with the `exec` package.

```go
// EXIF keys.
const (
	EXIFComment = "Comment"
	// Perhaps others...
)

func writeEXIF(filename, key, value string) error {
	if _, err := exec.Command(
		"exiftool",
		fmt.Sprintf(`-%s="%s"`, key, value),
		filename,
	).Output(); err != nil {
		return fmt.Errorf("exiftool failed: %w", err)
	}
	return nil
}

func getEXIF(filename, key string) ([]byte, error) {
	out, err := exec.Command(
		"exiftool", "-b",
		fmt.Sprintf("-%s", key),
		filename,
	).Output()
	if err != nil {
		return nil, fmt.Errorf("exiftool failed: %w", err)
	}
	if len(out) < 2 {
		return nil, fmt.Errorf("key empty")
	}
	return out[1 : len(out)-1], nil
}
```

We could get smarter with this, but this covers the basic use case of getting and setting a single EXIF attribute.
Note that because of how `exiftool` works, `writeEXIF` will overwrite `filename.JPG` create a `filename.JPG_original` file as a backup.

## Building an Image Checksum
Our digital signature needs some message to encrypt.
Generally, and especially for RSA, encrypting a hash of the data is the way.

```go
import (
	// ...
	
	// Remember to add all these packages for additional image support!
	"image"
	_ "image/jpeg"
	_ "image/png"
)

func uint32SliceToBytes(u []uint32) []byte {
	b := make([]byte, len(u)*4)
	for index, num := range u {
		binary.LittleEndian.PutUint32(b[index*4:index*4+4], num)
	}
	return b
}

func getImageDigest(filename string) ([]byte, error) {
	f, err := os.Open(filename)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	img, _, err := image.Decode(f)
	if err != nil {
		return nil, err
	}

	// Get all raw pixels.
	var pix []uint32
	for x := 0; x < img.Bounds().Dx(); x++ {
		for y := 0; y < img.Bounds().Dy(); y++ {
			r, g, b, a := img.At(x, y).RGBA()
			pix = append(pix, r, g, b, a)
		}
	}

	b := sha512.Sum512(uint32SliceToBytes(pix))
	return b[:], nil
}
```

Fantastic. 
`getImageDigest()` will let us take any arbitrary JPEG or PNG and get a SHA512 hash of the pixel data.
If any pixel is modified, the resulting hash will be different.

## Signing an Image
We need to sign the checksum, then insert it into the "Comment" EXIF field.

```go
func getRSAKey(filename string) (*rsa.PrivateKey, []byte, error) {
	f, err := os.Open(filename)
	if err != nil {
		return nil, nil, err
	}
	defer f.Close()

	// Remember, ioutil is deprecated in Go 1.16!
	b, err := io.ReadAll(f)
	if err != nil {
		return nil, nil, err
	}

	block, rest := pem.Decode(b)
	key, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		return nil, nil, err
	}
	return key, rest, nil
}

func sign(filename, rsaPath string) error {
	digest, err := getImageDigest(filename)
	if err != nil {
		return err
	}
	
	key, _, err := getRSAKey(rsaPath)
	if err != nil {
		return err
	}

	encrypted, err := rsa.SignPKCS1v15(rand.Reader, key, crypto.SHA512, digest)
	if err != nil {
		return nil
	}

	publicKey, err := x509.MarshalPKIXPublicKey(&key.PublicKey)
	if err != nil {
		return nil
	}
	metadata, err := json.Marshal(Metadata{
		PublicKey: publicKey,
		Signature: encrypted,
	})
	if err != nil {
		return nil
	}

	return writeEXIF(filename, EXIFComment, string(metadata))
}
```

Fortunately, the Go standard library has wonderful cryptography libraries.
Most of the work is done by the function `rsa.SignPKCS1v15()` function, with the surrounding code doing some data reading and transformation.

Let's sign an image and check out our new metadata.
![](https://inahga-public.s3-us-west-2.amazonaws.com/sign1.png)

## Verifying an Image
We're almost there.
Just need code to verify the image metadata is legitimate.

```go
func formatFingerprint(f string) string {
	for i := 2; i < len(f); i += 3 {
		f = f[:i] + ":" + f[i:]
	}
	return f
}

func verify(filename string) error {
	comment, err := getEXIF(filename, EXIFComment)
	if err != nil {
		return err
	}

	var metadata Metadata
	if err := json.Unmarshal(comment, &metadata); err != nil {
		return err
	}

	pkix, err := x509.ParsePKIXPublicKey(metadata.PublicKey)
	if err != nil {
		return err
	}
	key, ok := pkix.(*rsa.PublicKey)
	if !ok {
		return fmt.Errorf("unknown key format %T", key)
	}

	digest, err := getImageDigest(filename)
	if err != nil {
		return err
	}

	if rsa.VerifyPKCS1v15(key, crypto.SHA512, digest, metadata.Signature); err != nil {
		return fmt.Errorf("validation failed: %w", err)
	}

	fingerprint := md5.Sum(metadata.PublicKey)
	fmt.Printf("integrity check passed\nRSA public key fingerprint: %s\n",
		formatFingerprint(hex.EncodeToString(fingerprint[:])))

	return nil
}
```
Like with signing, `rsa.VerifyPKCS1v15()` is doing the heavy lifting.
It will decrypt the signature, and compare it against an expected 

Let's try it on the image we just signed.
![](https://inahga-public.s3-us-west-2.amazonaws.com/verify1.png)

Notice that we're outputting the public key fingerprint.
The viewer will need to manually ensure that this public key in fact belongs to the artist, until we come up with some kind of public key verification.

Let's throw this image in an editor and modify a single pixel.
![](https://inahga-public.s3-us-west-2.amazonaws.com/verify2.png)

Glorious. The change is imperceptible to the eye, but our tool can still detect it.

## Summary
Behind the scenes, I wrapped up this program in a nice CLI.
You can try it out for yourself.
```
go get github.com/inahga/inahgohno/cmd/imgsig
```

[Here's](https://inahga-public.s3-us-west-2.amazonaws.com/IMG_E0554_signed.JPG) a signed image to try it on.
I signed it with the public key `89:95:57:0f:fe:d0:37:65:a3:00:20:00:1b:cf:91:5d`.

Hopefully this can spur some thoughts on better means of verifying image authenticity.
Art being co-opted is a huge problem, one that causes unfair backlash, grief, and loss of reputation for artists.
