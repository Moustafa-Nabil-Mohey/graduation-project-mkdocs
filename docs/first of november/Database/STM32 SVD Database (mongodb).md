## Imports Overview
 * These imports provide all required functionality:
  - OS, RE, IO, COPY: utility and file operations
  - xml.etree.ElementTree: parsing SVD XML files
  - pymongo: MongoDB database connectivity
  - fitz (PyMuPDF): extracting text and images from PDF
  - pytesseract + PIL: OCR for unreadable PDF pages

??? info "Imports Overview" 

    * These imports provide all required functionality:
        - OS, RE, IO, COPY: utility and file operations
        - xml.etree.ElementTree: parsing SVD XML files
        - pymongo: MongoDB database connectivity
        - fitz (PyMuPDF): extracting text and images from PDF
        - pytesseract + PIL: OCR for unreadable PDF pages

```py title="code" linenums="1"
import os, sys, re, io, copy
import xml.etree.ElementTree as ET
from pymongo import MongoClient
import fitz, pytesseract
from PIL import Image
```

## MongoDB Connection
  This function connects to MongoDB Atlas, verifies connectivity, and returns the collection object used for inserting documents.
  
```py title="code" linenums="1"
def connect():
    c = MongoClient(MONGO_URI, tls=True)
    c.admin.command("ping")
    return c[DB][COL]
```
## PDF Base Address Extraction
  This function reads the reference manual PDF and extracts base addresses for peripherals. OCR is automatically used when a PDF page contains almost no readable text.

```py title="code" linenums="1"
def extract_pdf_bases(path):
    if not os.path.exists(path): return {}
    txt = ""
    for pg in fitz.open(path):
        t = pg.get_text()
        if len(t.strip()) < 40:
            img = Image.open(io.BytesIO(pg.get_pixmap(dpi=200).tobytes("png")))
            t = pytesseract.image_to_string(img)
        txt += t + "\n"
    bases = {}
    for m in re.finditer(r"([A-Z][A-Z0-9_]+)[^\w]{0,10}(0x[0-9A-Fa-f]{6,8})", txt):
        bases[m.group(1)] = m.group(2).upper()
    return bases
```
## Peripheral Expansion (Handling derivedFrom)
  Some peripherals inherit structure from others using SVD's derivedFrom. This function recursively expands inherited peripherals and merges fields correctly.

```py title="code" linenums="1"
def expand(peripherals, name, seen=None):
    if name not in peripherals: return None
    seen = seen or set()
    if name in seen: return None
    seen.add(name)
    node = peripherals[name]["raw"]
    base = peripherals[name]["derived"]
    if base:
        bnode = expand(peripherals, base, seen)
        if bnode is not None:
            m = copy.deepcopy(bnode)
            for ch in node:
                if ch.tag in ["registers","clusters"]:
                    for old in m.findall(ch.tag):
                        m.remove(old)
                m.append(copy.deepcopy(ch))
            m.find("name").text = node.findtext("name")
            m.find("baseAddress").text = node.findtext("baseAddress", m.findtext("baseAddress","0x0"))
            return m
    return node
```
## SVD Parsing 
  This is the main logic that reads SVD peripherals, registers, clusters, and fields. It calculates hex addresses, merges PDF bases, and produces MongoDB-ready documents.

```py title="code" linenums="1"
def parse_svd(path, pdf_bases):
    tree = ET.parse(path)
    root = tree.getroot().find("peripherals")
    per = {p.findtext("name"):{"raw":p,"derived":p.get("derivedFrom")} for p in root}
    docs = []

    def add(reg, P, desc, bint, bhex, pref=""):
        rname = pref + reg.findtext("name","").upper()
        off = int(reg.findtext("addressOffset","0x0"),0)
        hexaddr = hex(bint+off).upper()
        fields = reg.find("fields")
        if fields is None:
            docs.append({"PERIPHERAL":P,"DESCRIPTION":desc,"BASEADDRESS":bhex,
                         "REGISTER":rname,"REGISTER_DESCRIPTION":reg.findtext("description",""), 
                         "ADDRESSOFFSET":hex(off).upper(),"RESETVALUE":reg.findtext("resetValue","0"),
                         "HEXADDRESS":hexaddr,"FIELD":None,"FIELD_DESCRIPTION":None,
                         "BITOFFSET":None,"BITWIDTH":None})
        else:
            for f in fields.findall("field"):
                docs.append({"PERIPHERAL":P,"DESCRIPTION":desc,"BASEADDRESS":bhex,
                             "REGISTER":rname,"REGISTER_DESCRIPTION":reg.findtext("description",""), 
                             "ADDRESSOFFSET":hex(off).upper(),"RESETVALUE":reg.findtext("resetValue","0"),
                             "HEXADDRESS":hexaddr,"FIELD":f.findtext("name",""), 
                             "FIELD_DESCRIPTION":f.findtext("description",""), 
                             "BITOFFSET":int(f.findtext("bitOffset","0")), 
                             "BITWIDTH":int(f.findtext("bitWidth","0"))})

    for name in per:
        node = expand(per, name)
        if node is None: continue
        P = node.findtext("name","N/A").upper()
        base = pdf_bases.get(P, node.findtext("baseAddress","0x0"))
        bint = int(base,0); bhex = hex(bint).upper()
        desc = node.findtext("description","")

        regs = node.find("registers")
        if regs:
            for r in regs.findall("register"): add(r,P,desc,bint,bhex)

        for cl in node.findall(".//cluster"):
            cpre = cl.findtext("name","").upper()+"_"
            coff = int(cl.findtext("addressOffset","0x0"),0)
            for r in cl.findall("register"): add(r,P,desc,bint+coff,bhex,cpre)

    return docs
```
## MongoDB Insertion Workflow
  The main() function runs the entire pipeline: connects to DB, extracts PDF bases, parses SVD, clears old MongoDB documents, and inserts the new structured dataset.

```py title="code" linenums="1"
 def main():
    col = connect()
    bases = extract_pdf_bases(PDF)
    docs = parse_svd(SVD, bases)
    col.delete_many({})
    col.insert_many(docs)
    print("DONE:", len(docs),"records uploaded.")

 if __name__ == "__main__":
    main()
 database example + screenshot :-
 Collection: stm32_registers_full
 {
  "PERIPHERAL": "FLASH",
  "DESCRIPTION": "Flash memory control registers",
  "BASEADDRESS": "0X40023C00",
  "REGISTER": "ACR",
  "REGISTER_DESCRIPTION": "Flash access control register",
  "ADDRESSOFFSET": "0X0",
  "RESETVALUE": "0X20",
  "HEXADDRESS": "0X40023C00",
  "FIELD": "LATENCY",
  "FIELD_DESCRIPTION": "Latency",
  "BITOFFSET": 0,
  "BITWIDTH": 3
 }
```
<img src="../../Database/mongoDB.png" alt="MongoDB Logo" width="4000" 
     style="display:block; margin:auto; border:2px solid #4c7cafff; border-radius:8px; padding:5px;">


