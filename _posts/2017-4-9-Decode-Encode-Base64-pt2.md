---
layout: post
title: String Encoding and Decoding - Base64 - Part 2
---

I wasn't intending on making this a 2 part post ([Part 1](http://codeandkeep.com/Decode-Encode-Base64/)), but I couldn't leave the code alone.  I mentioned in part 1 that additional encoding options could be added, and that's exactly what I have done here.
<br>

We will be using the same .net classes as before [System.Text.Encode] and [System.Convert].
The really nice thing about the [Text.Encode] class is that the different types of encoding all live here, are named appropriately, AND they all contain the GetBytes() and GetString() methods.

This really makes updating these functions easy.
<br>

### Encode
----
```powershell
function ConvertTo-EncodedString {
  [CmdletBinding()] Param(
    [Parameter(
      Mandatory=$true,
      Position=0,
      ValueFromPipeline=$true
    )]
    [String[]]$String,

    [Parameter(Position=1)]
    [ValidateSet(
      'ASCII',
      'Unicode',
      'UTF32',
      'UTF8',
      'UTF7'
    )]
    [String]$Encoding='Unicode'
  )
  Begin{} 
  Process{
    foreach($str in $String){
      Try{
        [Byte[]]$uniBytes=[Text.Encoding]::$Encoding.GetBytes($str)
        [String]$encode=[Convert]::ToBase64String($uniBytes)
        Write-Output $encode
      }Catch{
        Write-Error $_
      }
    }
  }
  End{}
}
```


### Decode
----
```powershell
function ConvertFrom-EncodedString {
  [CmdletBinding()]
  Param(
    [Parameter(
      Mandatory=$true,
      Position=0,
      ValueFromPipeline=$true
    )]
    [String[]]$String,

    [Parameter(Position=1)]
    [ValidateSet(
      'ASCII',
      'Unicode',
      'UTF32',
      'UTF8',
      'UTF7'
    )]
    [String]$Encoding='Unicode'
  )
  Begin{}
  Process{
    foreach($str in $String){
      Try{
        [Byte[]]$baseBytes=[Convert]::FromBase64String($str)
        [String]$decode=[Text.Encoding]::$Encoding.GetString($baseBytes)
        Write-Output $decode
      }Catch{
        Write-Error $_
      }
    }
  }
  End{}
}
```
<br>
### Examples
```powershell
# Puts your encoded string on your Desktop
ConvertTo-EncodedString 'Super secret message' > ~/Desktop/Nothing_to_see_here.txt

# Decode the secret message
cat ~/Desktop/Nothing_to_see_here.txt | ConvertFrom-EncodedString 
```
```powershell
$secret='Hello', 'World' | ConvertTo-EncodedString -Encoding UTF32

ConvertFrom-EncodedString -Encoding UTF32 -String $secret
```
<br>

#### Additional Comments
----
As you can see, this was made incredibly easy be restricting the values that are allowed for the 'Encoding' parameter to the exact values that are used in the [Text.Encoding] class.  

Another change worth noting is that both functions will now accept multiple strings.  

The errors generated by the methods used in these functions are terminating errors, so they will get caught without continuing and filling up the screen with red text.
<br>

Something that I could add, is support for multiple Encoding values.  This would essentially provide a simple way to double or triple encode/decode strings with the same, or even different encodings.  This seems like overkill to me, but maybe I will cave and add it in the next post.

<br>
*Thanks for reading,*

PS> exit