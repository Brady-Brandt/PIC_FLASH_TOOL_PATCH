# PIC_FLASH_TOOL Patch
Provides a patched binary of a tool we use to program our PIC processors in my embedded class. Download it [here](https://github.com/Brady-Brandt/PIC_FLASH_TOOL_PATCH/releases/tag/v1)

# Problem 
The original flash tool would only accept files containing the string "hex" but the IDE we used to develop and compile assembly
would produce .HEX files, so you had to change the file extension to .hex in order for the flash tool to accept the file.
I got tired of doing this and I wanted to test out my reverse engineering skills, so I began working on a solution.

# Setup
I used the disassembler [dnSpy](https://github.com/dnSpy/dnSpy) to reverse engineer the binary.
dnySpy decompiles .NET code and allows you to view the decompiled code in C#, Visual Basic, or IL (basically .NET assembly).
C# is by far the most readable out of these three so thats what I used. I used the search feature in dnSpy to search for strings
that I knew were in the program and that lead me to a class called form1, which holds pretty much the entire program. From here, I searched for
the string "hex" which lead me to this function.

```c#
private void timer_button_Click(object sender, EventArgs e)
		{
			OpenFileDialog openFileDialog = new OpenFileDialog();
			openFileDialog.ShowDialog();
			this.filespath.Text = openFileDialog.FileName;
			if (this.filespath.Text.Contains("hex"))
			{
				this.file_selected.Text = "Complete: Hex file has been selected";
				this.file_selected.ForeColor = Color.Black;
				if (this.reset_text.Text == "Complete: Micro Reset/Cleared")
				{
					this.send_button.Enabled = true;
					this.programming.Text = "Programming =";
				}
			}
			else
			{
				MessageBox.Show("Ensure that the file you have selected is of type hex", "File Type Warning");
			}
			this.UpdateStatus("File has been selected");
		}
```
As you can see this function uses the .contains method to check whether or a not a string is a hex file. It doesn't even have to end with .hex just contain the word hex.
So files like hex.asm would be accepted but the file blink.HEX gets rejected. I just needed make a couple changes to the if statement condition to get it to accept any .hex file ignoring case.
## IL Assembly
Unfortunately, I can't just edit the C# code. The decompilation and recompilation wouldn't work, but I could edit the binary's bytecode. The ILASM of this if statement looks like this.
```asm
/* 0x00001C01 72C3090070   */ IL_0029: ldstr     "hex"
/* 0x00001C06 6F7800000A   */ IL_002E: callvirt  instance bool [mscorlib]System.String::Contains(string)
/* 0x00001C0B 2C55         */ IL_0033: brfalse.s IL_008A
```
## Editing IL Assembly
The first thing I do is change the string "hex" to ".hex" to ensure a proper file extension. Then I replace the "contains" method with "endswith", but I use the version that takes in a StringComparsion enum as a parameter. I do this so I can perform an ignore case string comparison. Finally I need to pass the value of the enum to the function call. Now for some reason I couldn't use the actual enum, I kept getting a TypeNotFound exception, but luckily enums are just constant integers. According to the StringComparison enum documentation the enum field I should use is OrdinalIgnoreCase and it has a value of 5. So our new assembly would look like this.
```asm
/* 0x00001C01 72C3090070   */ IL_0029: ldstr     ".hex"
/* 0x00001C06 2005000000   */ IL_002E: ldc.i4    5
/* 0x00001C0B 6F9100000A   */ IL_0033: callvirt  instance bool [mscorlib]System.String::EndsWith(string, valuetype [mscorlib]System.StringComparison)
```
## Wrapping it up
After saving the changes, the "new" binary will now accept .HEX files and it will still continue to accept .hex files. Now I don't have to rename the file everytime I want to compile and upload code with MPLAB.

# Copyright
This original binary is not my own work. I simply inserted a few lines of bytecode to make the program better. All rights go to the original author Nathan Zimmerman @ 2012. 

