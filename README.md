# powershell-in-arduino-IDE
I needed to hotpatch the assembly .elf code before arduino converts it to .hex and uploads it to the microcontroller so I created this.
It will allow you to place powershell code in your main file, it will be ran after the compiler generated the .elf file.


### WARNING
This is an advanced tool. It is **HIGHLY** recommended to set `build.verbose` to `true` in `%APPDATA%/../Local/Arduino15/preferences.txt`.



### HOW TO INSTALL
Add this line to the end of platform.txt: (this file can usually be found in `%APPDATA%/../Local/Arduino15/packages/arduino/hardware/avr/1.6.x`)

```
recipe.hooks.objcopy.preobjcopy.1337.pattern=powershell $buildPath="{build.path}";$projectName="{build.project_name}";Select-String ('\/\*\*[ \t]*POWERSHELL([\s\S]+?)\*?\*\/') -input ([IO.File]::ReadAllText("{build.path}/sketch/{build.project_name}.cpp")) -AllMatches | Foreach {try{Invoke-Expression $_.Matches.Groups[1].Value}catch{Write-Error ("$($_.Exception.GetType().FullName) $($_.Exception.Message)")}}
```


### HOW TO USE

In your **main** file, you can add a multiline comment like so:

```
/**  POWERSHELL

[System.IO.File]::ReadAllBytes("$buildPath/$projectName.elf")

 */
```

This will print the path to the .elf path of your assembly in the console.


### Example usage:

Here is a sample code that I wrote. It quickly writes `r17` and `r16` to `PORTD` when `TIMER2_COMPA` and `TIMER2_COMPB` interrupts are triggered. It places this assembly code directly into the jmp table at the beginning of the program memory (which is afaik impossible without modding the arduino IDE like I did, please let me know if I'm wrong)


Note 1: I couldn't get this code to compile on arduino avr boards versions 1.6.12 and up.

Note 2: The powershell code relies on external interrupt 0 and 1 jumps both pointing to the same address (\_\_bad\_interrupt by default when the ISR functions for these are not defined). This is entirely because I'm lazy. 

```
register byte onB asm(r16);
register byte offB asm(r17);


...


/** POWERSHELL

$bytes  = [System.IO.File]::ReadAllBytes("$buildPath/$projectName.elf")
$offset = 0
$continue = $true
while ($continue) {
    if (($bytes[$offset] -eq 0x0C) -and ($bytes[$offset + 1] -eq 0x94) -and ($bytes[$offset + 2] -eq $bytes[$offset + 6]) -and ($bytes[$offset + 3] -eq $bytes[$offset + 7])) {
        #match found, first bit of 2 jump sequences to the same area, aka bad interrupt, so we found adress 0x04
        $continue = $false
    } else {
        $offset = $offset + 1
    }
    if ($offset -eq 1000) {
        throw "cannot find adress 0x04"
    }
}
$offset = $offset + 24

#COMPA, TURNS OFF PWM
#out 0x05, r17 = write r17 to PORTD
$bytes[$offset++] = 0x15
$bytes[$offset++] = 0xb9
#reti
$bytes[$offset++] = 0x18
$bytes[$offset++] = 0x95
#COMPB, TURNS ON PWM
#out 0x05, r16 = write r16 to PORTD
$bytes[$offset++] = 0x05
$bytes[$offset++] = 0xb9
#reti
$bytes[$offset++] = 0x18
$bytes[$offset++] = 0x95

[System.IO.File]::WriteAllBytes("$buildPath/$projectName.elf", $bytes)

echo "HEX PATCH SUCCESS! (I hope)"

 */
```
