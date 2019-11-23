```python
import numpy as np
#pip install bitstring
from bitstring import BitArray
import math
import os

```

### Store with RAID6


```python
#number of data storage nodes. except parity nodes.
nbOfDirectories = 5
#name of the file to store
accessedFile = "big3"
#Current path
path="C:\\Users\\PANDA\\RAID6"
#Call function to segment the file in nbOfDirectories equal parts
buildFile(accessedFile, path, nbOfDirectories)
```

    Directory 0 created
    Directory 1 created
    Directory 2 created
    Directory 3 created
    Directory 4 created
    Directory 5 created
    Directory 6 created
    


```python
def buildFile(accessedFile, path, nbOfDirectories):
    dirlist = os.listdir(path)
    #get list of directories
    dirlist = [i for i in dirlist if (os.path.isdir(path + "\\" + i) and i[0] == "D")]
    missingDir = []
    #create list of missing directories
    if len(dirlist) != nbOfDirectories + 2:
        for i in range(nbOfDirectories + 2):
            if("D" + str(i) not in dirlist):
                missingDir.append(i)
    
    for i in missingDir:
        os.mkdir("D" + str(i))
        print("Directory", i, "created")
    
    file = open(accessedFile + '.txt', 'rb')
    temp = file.read().splitlines()
    t = ''
    #read the file bytes by bytes and store it in a tab
    tab = []
    for line in temp:
        for char in line:
            tab.append(chr(char).encode("utf-8"))
        tab.append("\r".encode("utf-8"))
   #erase the last value in the tab wich is an empty byte
    tab = tab[:-1]
    file.close()
    t = []
    #size of each segment
    # split the file into the differents data stripes folders
    #each line in t will be store in a différent folder (t)
    split = int(len(tab)/nbOfDirectories)
    i = 0
    curr = 0
    while i <= len(tab):
        if curr < (len(tab)/nbOfDirectories - split) * nbOfDirectories:
            t.append(tab[i:i+split+1])
            i += split + 1
        else:
            t.append(tab[i:i+split])
            i += split
        curr += 1
    p = [0 for i in range(len(t[0]))]
    q = [0 for i in range(len(t[0]))]
    for i in range(len(t)):
        for j in range(len(t[i])):
            p[j] = p[j] ^ int.from_bytes(t[i][j], "big")
            q[j] ^= (2 ** (nbOfDirectories - i - 1)) * int.from_bytes(t[i][j], "big")
    for d in range(nbOfDirectories+2):
        try:
            os.mkdir("D" + str(d))
        except:
            pass
    for d in range(nbOfDirectories):
        f = open("D" + str(d) + "/" + accessedFile + "_" + str(d) +"_0.txt", "w")
        for i in t[d]:
            f.write(i.decode("utf-8"))

        f.close()
    f = open("D" + str(nbOfDirectories) + "/" + accessedFile + "_" + str(nbOfDirectories) + "_p.txt", "w")
    for i in p:
        f.write(str(i) + " ")
    f.close()
    f = open("D" + str(nbOfDirectories+1) + "/" + accessedFile + "_" + str(nbOfDirectories+1) + "_q.txt", "w")
    for i in q:
        f.write(str(i) + " ")
    f.close()
```

### Access Files


```python
#name of the file we want to access
wantToAccess = "NewFile"
```


```python
def writeToFile(tab, file):
    f = open(file, "w")
    for i in tab:
        f.write(str(i))
    f.close()

myFile = ""
#get though the folders and each pieces of the file and concatenante it into one file
for i in range(nbOfDirectories+2):
    try:
        file = open("D"+str(i) + "/" + wantToAccess + "_" + str(i) + '_0.txt', 'rb')
        tmp = file.read(1)
        while tmp != b"":
            myFile += str(tmp.decode("utf-8"))
            tmp = file.read(1)
        file.close()
    except:
        pass
    
writeToFile(myFile, wantToAccess+"New3.txt")
```

### Detecting failure


```python
#pip install bitstring
from bitstring import BitArray
import math
import os
```


```python
def areWeMissingSomething(nbOfDirectories, path):
    dictPresentFile,fileInDir,dictMissingFile, missingSomething=getMissingFileAndFolder(nbOfDirectories, path)
    #Not missing anything! Nothing to recover.
    if not missingSomething:
        print("Nothing to recover!")
        return 0
    print("We are missing something! Let's recover!")
    #recover()
    return 1
```


```python
areWeMissingSomething(nbOfDirectories, path)
```

### We lost something


```python
def getMissingFileAndFolder(nbOfDirectories, path):
    #list the directories 
    dirlist = os.listdir(path)
    dirlist = [i for i in dirlist if (os.path.isdir(path + "\\" + i) and i[0] == "D")]
    missingDir = []
    if len(dirlist) != nbOfDirectories + 2:
        for i in range(nbOfDirectories + 2):
            if("D" + str(i) not in dirlist):
                missingDir.append(i)
    #create missing directories
    for i in missingDir:
        os.mkdir("D" + str(i))
        print("Directory", i, "created")
    
    #Find files present and missing in directories available
    dictPresentFile = dict()
    fileInDir = [dict() for i in range(nbOfDirectories + 2)]
    
    for i in range(nbOfDirectories+2):
        if i not in missingDir:       
            pathToCheck = path + "\\D" + str(i)
            for f in os.listdir(pathToCheck):
                if os.path.isfile(os.path.join(pathToCheck, f)):
                    file = f.split("_")[-3]
                    if file not in dictPresentFile.keys():
                        dictPresentFile[file] = []
                    dictPresentFile[file].append(i)
                    fileInDir[i][file] = f
    #Directories where file part is not available
    missingSomething = False
    dictMissingFile = dict()
    for i in dictPresentFile.keys():
        dictMissingFile[i] = [j for j in range(nbOfDirectories + 2) if j not in dictPresentFile[i]]
        missingSomething += len(dictMissingFile[i]) > 0
        if not len(dictMissingFile[i]) > 0:
            del dictMissingFile[i]
    
    return dictPresentFile,fileInDir, dictMissingFile, missingSomething
```


```python
def recover(nbOfDirectories, path):
    
    dictPresentFile,fileInDir,dictMissingFile, missingSomething=getMissingFileAndFolder(nbOfDirectories, path)    
    
    #Not missing anything! Nothing to recover.
    if not missingSomething:
        print("Nothing to recover!")
        return
    
    
    #for each missing file in directories
    for i in dictMissingFile.keys():
        
        fileName = missingFile = i
        files = ["D" + fileInDir[j][i].split("_")[-2] + "\\" + fileInDir[j][i] 
             for j in range(len(fileInDir)) 
             if i in fileInDir[j].keys()
            ]
        
        extension = "." + files[0].split(".")[-1]
        part = files[0].split("_")[-1].split(".")[0]
        #Link between x of Dx and index in files tab
        indexes = [ j - sum([1 for k in dictMissingFile[i] 
                             if int(fileInDir[j][i].split("_")[-2]) > k])
                       if i in fileInDir[j].keys()
                       else -1
                    for j in range(len(fileInDir)) 

                  ]
        print("Recovering:", missingFile + extension, "- part:", part)
        #Load data of files present of name i
        data = []
        for f in files: 
            if f.split("_")[-1][0] not in ["p","q"]:
                data.append(read(f, ""))
            else:
                data.append(read(f, " "))
                
        #If only missing one part (use only p or q)
        if len(dictMissingFile[i]) == 1:
            #If p is missing
            if nbOfDirectories in dictMissingFile[i]: txt = generateP(data[:-1])
            #If q is missing
            elif nbOfDirectories+1 in dictMissingFile[i]: txt = generateQ(data[:-1])
            else:
                txt = recoverOneFileUsingP(data, nbOfDirectories, dictMissingFile[i], indexes)
            path = "D" + str(dictMissingFile[i][0]) + "\\" + missingFile + "_" \
                       + str(dictMissingFile[i][0]) + "_"  + \
                       (part if dictMissingFile[i][0] < nbOfDirectories else \
                        "p"  if dictMissingFile[i][0] == nbOfDirectories else \
                        "q") + extension
            if dictMissingFile[i][0] >= nbOfDirectories:
                write(path, txt, " ")
            else:
                write(path, txt)
        #Otherwise we need to solve the system on p and q
        else : 
            #If missing p and q
            if nbOfDirectories in dictMissingFile[i] and nbOfDirectories+1 in dictMissingFile[i]:
                a = generateP(data) 
                b = generateQ(data)
            #If missing p and something else (not q)
            elif nbOfDirectories in dictMissingFile[i]:
                a = recoverOneFileUsingQ(data, nbOfDirectories, dictMissingFile[i], indexes)
                data.insert(0,a)
                b = generateP(data[:-1])
            #If missing q and something else (not p)
            elif nbOfDirectories + 1 in dictMissingFile[i]:
                a = recoverOneFileUsingP(data, nbOfDirectories, dictMissingFile[i], indexes)
                data.insert(0,a)
                b = generateQ(data[:-1])
            else:
                a,b = recoverTwoFiles(data,nbOfDirectories, dictMissingFile[i], indexes)
            for elem in [0,1]:
                if elem == 0: txt = a
                else: txt = b
                path = "D" + str(dictMissingFile[i][elem]) + "\\" + missingFile + "_" \
                           + str(dictMissingFile[i][elem]) + "_"  + \
                           (part if dictMissingFile[i][elem] < nbOfDirectories else \
                            "p"  if dictMissingFile[i][elem] == nbOfDirectories else \
                            "q") + extension
                if dictMissingFile[i][elem] >= nbOfDirectories:
                    write(path, txt, " ")
                else:
                    write(path, txt)
        print(fileName + extension, "recovered!")
```


```python
def generateP(data):
    p = [0 for i in range(len(data[0]))]
    for i in range(len(data)):
        for j in range(len(data[i])):
            p[j] = p[j] ^ ord(data[i][j])
    return p

def generateQ(data):
    q = [0 for i in range(len(data[0]))]
    for i in range(len(data)):
        for j in range(len(data[i])):
            q[j] = q[j] * 2 ^ ord(data[i][j])
    return q
```


```python
def recoverOneFileUsingP(data, nbOfDirectories, missingDir, indexes):
    firstFile = []
    for col in range(len(data[0])):
        pCoeff = [1 for i in range(nbOfDirectories)]
        pResult = 0
        for d in range(nbOfDirectories):
            if d not in missingDir:
                #in case we don't have exact same length for every file
                if col < len(data[indexes[d]]): 
                    pResult = pResult ^ pCoeff[d] * ord(data[indexes[d]][col])
                    pCoeff[d] = 0
        
        pResult ^= int(data[-1 - (nbOfDirectories + 1 not in missingDir)][col])
        if pResult != 0:
            firstFile.append(chr(pResult))
            
    return firstFile

def recoverOneFileUsingQ(data, nbOfDirectories, missingDir, indexes):
    firstFile = []
    for col in range(len(data[0])):
        qCoeff = [2 ** (nbOfDirectories - j - 1) for j in range(nbOfDirectories)]
        qResult = 0
        for d in range(nbOfDirectories):
            if d not in missingDir:
                #in case we don't have exact same length for every file
                if col < len(data[indexes[d]]): 
                    qResult = qResult ^ qCoeff[d] * ord(data[indexes[d]][col])
                    qCoeff[d] = 0
        
        qResult ^= int(data[-1][col])
        if qResult != 0:
            firstFile.append(chr(qResult >> int(nbOfDirectories - missingDir[0] - 1)))
    return firstFile
```


```python
def recoverTwoFiles(data, nbOfDirectories, missingDir, indexes):
    firstFile = []
    secondFile = []
    for col in range(len(data[0])):
        pCoeff = [1 for i in range(nbOfDirectories)]
        qCoeff = [2 ** (nbOfDirectories - j - 1) for j in range(nbOfDirectories)]
        pResult = 0
        qResult = 0
        for d in range(nbOfDirectories):
            if d not in missingDir:
                if col < len(data[indexes[d]]): 
                    qResult = qResult ^ qCoeff[d] * ord(data[indexes[d]][col])
                    pResult = pResult ^ pCoeff[d] * ord(data[indexes[d]][col])
                    qCoeff[d] = 0
                    pCoeff[d] = 0
                    
        if nbOfDirectories not in missingDir:
            pResult ^= int(data[-2][col])
        if nbOfDirectories+1 not in missingDir:
            qResult ^= int(data[-1][col])
            
        a,b = solve(missingDir, pResult, qResult,nbOfDirectories)
        #If file is not a p or q we transform into Char
        #Otherwise we keep them as nb
        if a != 0: firstFile.append(chr(a))
        if b != 0: secondFile.append(chr(b))
    return firstFile, secondFile
```


```python
def write(path, text, sep = ""):
    file = open(path, 'w')
    try:
        for i in text:
            file.write(str(i) + sep)
    except:
        print("Bip")
        pass
    finally:
        file.close

def read(file, sep = ""):
    file = open(file, 'rb')
    tmp = file.read(1)
    tab = []
    while tmp != b'':
        tab.append(tmp.decode("utf-8"))
        tmp = file.read(1)
    file.close()
    
    if sep != "":
        tmp = ""
        tab2 = []
        for i in tab:
            if i == sep:
                tab2.append(tmp)
                tmp = ""
            else:
                tmp += i
        tab = tab2
    return tab


# Solve system of :
# x xor y = newP
# a*x xor b*y = newQ
# where a and b have the form 2 ** n with n one of the missing directory
def solve(missing, newP, newQ, nbOfDirectories):
    #Calcul coeff a and b
    coeff = [2 ** (nbOfDirectories - j - 1) for j in missing]
    p = newP
    q = [int(i) for i in BitArray(hex=hex(newQ ^ coeff[-1] * newP)).bin]
    x = []
    tmp = []
    
    n = int(math.log(min(coeff),2))
    np = int(math.log(max(coeff),2)) 
    
    for i in range(0,n):
        q.pop()
    
    for i in range(n, np):
        x.insert(0,q.pop())
        tmp.insert(0,x[0])
    
    for i in range(np , 8+np):
        if len(q) > 0:
            x.insert(0,q.pop() ^ tmp.pop())
            tmp.insert(0, x[0])

        else:
            x.insert(0, 0 ^ tmp.pop())
            tmp.insert(0, x[0])
    
    tmp = 0
    for i in x:
        tmp = tmp*2 + int(i)
    x = tmp
    y = x ^ p
    return x,y
```


```python
recover(nbOfDirectories, path)
```
