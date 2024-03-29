---
title: "Whittling Rhino Licensing"
layout: post
---


Last week, I introduced a learning exercise called <a href="http://patrick.lioi.net/2013/01/03/code-whittling/">Code Whittling</a>.  This week, we'll walk through the process I took when whittling <a href="http://www.hibernatingrhinos.com/oss/rhino-licensing">Rhino Licensing</a>.

Rhino Licensing makes it easy for your application to enforce licenses.  If you're making a application that needs to check which features the user has paid for, whether a trial has expired, etc, this is about as easy as it gets.  I recently needed to do this for a client, and James Gregory's <a href="http://www.jagregory.com/writings/rhino-licensing/">Rhino Licensing tutorial</a> covers exactly the use case I cared about.  The tool, however, handles more than that use case.  I wanted to know how the basic idea worked, how a human-readable XML file could be tamper-proof, but I didn't care about the other supported use cases.

## The Whittle

I started by grabbing a copy of the <a href="https://github.com/hibernating-rhinos/rhino-licensing">Rhino Licensing source</a>.

GUI tools in support of a library don't usually catch my interest.  Rhino.Licensing.AdminTool was a UI tool I didn't need to work with for my use case, so it was the first to go.

Rhino Licensing can use trusted time servers like time.nist.gov when deciding whether a license's expiration date has been reached, since malicious users might subvert expiration dates by changing their local system clock to some future date.  This feature is important, but was irrelevant to me in the short term: I wanted to know how the license files themselves work, and cared less about the details of reliable timekeeping.  I removed the class <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/SntpClient.cs">SntpClient</a>, the array of trusted servers within <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/AbstractLicenseValidator.cs">AbstractLicenseValidator</a>, and everything that depended on them.

There are several exception types declared, which is useful for communicating why a license might fail validation, but they didn't help me in my quest to find out how the license files could be tamper-proof, so I phased them out in favor of generic Exception instances.  Likewise, logging is important, but it was just noise for my purposes, so I removed it.

The <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/LicenseType.cs">LicenseType</a> enum lists several kinds of licenses I was uninterested in.  I removed the ones I wasn't interested in along with their usages.

Rhino Licensing has a server-side component, I believe to coordinate the sharing of license files across multiple machines.  Although interesting, this doesn't have to do with the tamper-proofing of license file content, so I removed: LicensingService, ISubscriptionLicensingService, ILicensingService, DiscoveryHost, DiscoveryClient, and anything that depended on them.

<a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/AbstractLicenseValidator.cs">AbstractLicenseValidator</a> has two child classes: <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/LicenseValidator.cs">LicenseValidator</a> for processing a license XML file, and <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/StringLicenseValidator.cs">StringLicenseValidator</a> for processing a string of XML (for instance, you may be storing your license content in a database rather than a file).  I didn't need a refresher course on reading file content into a string, so I removed LicenseValidator and then collapsed StringLicenseValidator into the AbstractLicenseValidator base class.

## Results - Signing XML Documents

All this left me with a better understanding of what Rhino Licensing has to offer, while whittling the project down to the few files I *really* wanted to dig into: <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/LicenseGenerator.cs">LicenseGenerator</a>, and a now-much-much-smaller version of <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/AbstractLicenseValidator.cs">AbstractLicenseValidator</a>.  The first is responsible for generating the tamper-proof XML, and the second is responsible for parsing the tamper-proof XML and deciding whether it is legitimate.

James Gregory's <a href="http://www.jagregory.com/writings/rhino-licensing/">Rhino Licensing tutorial</a> first describes a prerequisite step for creating a private/public key pair:

```cs
class Program
{
  static void Main(string[] args)
  {
    var rsa = new RSACryptoServiceProvider(1024);
    File.WriteAllText("publicKey.xml", rsa.ToXmlString(false));
    File.WriteAllText("privateKey.xml", rsa.ToXmlString(true));
  }
}
```

The contents of these files are basically large numbers.  The numbers are related to each other, but a malicious user would not be able to guess the private one given only the public one.

<a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/LicenseGenerator.cs">LicenseGenerator</a> is given the private one as a string to its constructor.  LicenseGenerator then generates a string of XML given a few values.  The values will all appear in plain text in the XML: name, id, expirationDate, LicenseType, and an optional set of name/value pairs specific to your needs.  The private helper method CreateDocument produces an XmlDocument containing those inputs.

**Next, the magic happens.**  This simple XML document is completely open to tampering by a malicious user.  To combat our adversary, it supplements the document with an additional XML element produced by private helper method GetXmlDigitalSignature.  This method uses some .NET framework classes to take the original document and use the secret/private key to produce an encrypted version of the original weak XML document.  The new XML element basically contains yet another very large number, which corresponds with the original XML document content.  The resulting document is now tamper-proof: if a malicious user opens it and changes the expiration date to the year 2500, they would have to *also* change the big signature number to match, and *good luck guessing what big number corresponds with the new XML document*, since the malicious user doesn't have the private key necessary to create that number quickly.  To sum up, here's how to take an insecure XML document and make it tamper-proof with a signature:

```cs
public XmlDocument GetSignedXmlDocument()
{
  using (var rsa = new RSACryptoServiceProvider())
  {
    rsa.FromXmlString(privateKey);
  
    var doc = CreateInsecurePlainXmlDocument();
  
    //Secure doc from tampering:
    var signature = Signature(doc, rsa);
    doc.FirstChild.AppendChild(doc.ImportNode(signature, true));
  
    return doc;
  }
}

private static XmlElement Signature(XmlDocument doc, AsymmetricAlgorithm key)
{
  var signedDoc = new SignedXml(doc) { SigningKey = key };
  
  var reference = new Reference { Uri = "" };
  reference.AddTransform(new XmlDsigEnvelopedSignatureTransform());
  
  signedDoc.AddReference(reference);
  signedDoc.ComputeSignature();

  return signedDoc.GetXml();
}
```

Once we have one of these documents, how do we read and validate them?  <a href="https://github.com/hibernating-rhinos/rhino-licensing/blob/4d77e00bfa85e8d22f966b09e340be31476afac1/Rhino.Licensing/AbstractLicenseValidator.cs">AbstractLicenseValidator</a> had been whittled down to be little more than the methods ValidateXmlDocumentLicense() and TryGetValidDocument(string licensePublicKey, XmlDocument doc).  Contrary to its name, the main thing ValidateXmlDocumentLicense does is read the original inputs from the human-readable parts of the XML file.  TryGetValidDocument does the real work, the inverse of LicenseGenerator's digital "signature": read the special `<Signature>` element which contains our encrypted original document, and ask the .NET framework whether it matches the rest of the document's human-readable content:

```cs
private bool IsValidDocument(string licensePublicKey, XmlDocument doc)
{
    var rsa = new RSACryptoServiceProvider();
    rsa.FromXmlString(licensePublicKey);

    var nsMgr = new XmlNamespaceManager(doc.NameTable);
    nsMgr.AddNamespace("sig", "http://www.w3.org/2000/09/xmldsig#");

    var signedXml = new SignedXml(doc);
    var sig = (XmlElement)doc.SelectSingleNode("//sig:Signature", nsMgr);
    if (sig == null)
        return false;

    signedXml.LoadXml(sig);

    return signedXml.CheckSignature(rsa);
}
```

Creating tamper-proof yet human-readable XML files is no longer magical.  Rhino Licensing creates and validates tamper-proof license files by taking the original insecure version of the document and appending a special "content summary" element.  A malicious user would have to tamper with the human-readable part as well as the virtually-unguessable content summary in order to trick the validator, and as long as the private key is kept private, that's certainly good-enough.
