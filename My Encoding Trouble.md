# The issue
Encoding in Computer Science has a long history, a lot of stories. 
For me, encoding simply means to translate something from what the computer can understand to what I can understand.
It was not that important until one day I run into such a trouble.

I downloaded a csv from our Azure log space as a reference file. 
Then, it was loaded: 
```python
with APP_NAME_TABLE_PATH.open() as reference_file:
	app_name_file = pd.read_csv(reference_file)
	app_filtered = app_name_file[["AppId", "AppName"]]
```

But Python kept telling me: 
```python
KeyError: "['AppId'] not in index"
```

The reason was the strange column name:
```python
print(app_name_file.keys())

Output:
Index(['ï»¿AppId', 'ClusterId', 'AppName', 'SolutionArea', 'StaticByteSize',
       'FileSize', 'ModifiedDate [UTC]', 'SievoTenantId'],
      dtype='str')
```

Soon I learned the ï»¿ is called BOM (Byte Order Mark). It is used by computers to know how the text in the file should be decoded.

Then I tried this:
```python
with APP_NAME_TABLE_PATH.open() as reference_file:
	app_name_file = pd.read_csv(reference_file, encoding="utf-8")
	app_filtered = app_name_file[["AppId", "AppName"]]
```

The return surprised me
```python
ValueError: The specified reader encoding utf-8 is different from the encoding cp1252 of file
```

utf-8-sig doesn't work either! 
The ValueError disappears after using encoding="cp1252", but the BOM could not be removed. 
Claude concluded that the BOM is not at the file level, but at the text level.

Therefore, added a logic to remove the BOM:
```python
with APP_NAME_TABLE_PATH.open() as reference_file:
	app_name_file = pd.read_csv(reference_file, encoding="cp1252")
	app_name_file.columns = ["AppId" if "AppId" in x else x for x in app_name_file.columns]
	app_filtered = app_name_file[["AppId", "AppName"]]
```

# The final understanding
The final understanding arrived after some confused days. 
Actually, the issue was caused by the location of the encoding!

This code works correctly:
```python
with open(APP_NAME_TABLE_PATH, encoding="utf-8") as reference_file:
	app_name_file = pd.read_csv(reference_file)
	app_filtered = app_name_file[["AppId", "AppName"]]
```

Once I moved the encoding to the with open( ), there is no BOM issue, no cp1252!

WHY???

To my current understanding, Python's built-in open( ) uses a platform-dependent encoding. I use windows machine. cp1252 is the encoding standard created by Microsoft!
When there is no encoding in the open( ) function, it loaded the file with the Windows cp1252 encoding. 

To verify this hypothesis:
```python
with open(APP_NAME_TABLE_PATH) as reference_file:
    app_name_file = pd.read_csv(reference_file)

print(reference_file.encoding)

Ouput:
cp1252
```

```python
with open(APP_NAME_TABLE_PATH, encoding='cp1252') as reference_file:
    app_name_file = pd.read_csv(reference_file, encoding="utf-8") 

ValueError: The specified reader encoding utf-8 is different from the encoding cp1252 of file
```

```python
with open(APP_NAME_TABLE_PATH, encoding="cp1252") as reference_file:
    app_name_file = pd.read_csv(reference_file) 

print(app_name_file.keys())

Output:
Index(['ï»¿AppId', 'ClusterId', 'AppName', 'SolutionArea', 'StaticByteSize',
       'FileSize', 'ModifiedDate [UTC]', 'SievoTenantId'],
      dtype='str')
```

So, opening with cp1252 is the culprit.

CP1252 is a fixed-width, single-byte legacy encoding. It contains no byte order marker (BOM).
Therefore, when the file is opened with cp1252 encoding, the open( ) function simply 'translated' the 3 BOM bytes (EF, BB, BF) into 3 characters which look like ï»¿.

Pandas' read_csv( ) does powerful work with encoding. But, it cannot decode when there is no BOM, because the open( ) function turned the BOM markers into characters. 

# The lesson learned
When using Python's native open( ) function, always handle encoding during file handling! 
