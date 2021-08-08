TUBBY'S GUIDE TO ALL THINGS PORTAL 2 EXTERNAL MODDING

<Before we start, i will go through some 'alternative' options for executing commands and editing console variables in Portal 2>

1: The .cfg method

this is the WORST way of doing this, it relies on you having portal in focus at all times, it uses keypresses, it requires for the user 
to set binds before it can funtion, and it's limited to only game developer console limits.

this method involves setting a bind in the portal 2 developer console (accesable by the ` key, you must enable in settings)
this bind sets an uncommon key to be used during gameplay to execute a .cfg file in console.
(the bind used in previous versions of portal 2 chaos = "'bind ] "exec chaos.cfg"')

this relies on the fact that portal 2, and all (or most) of source engine games have an 'exec' console command,
that runs a .cfg file.

a .cfg file is just a list of console commands to run upon execution, and is just in the format of a text file.
.cfg files are used in source games, usually in the form of an autoexec.cfg file, this file automatically executes on the games launch

in order to use the exec command with a config file, it must be located in: steamapps\common\Portal 2\portal2\cfg (you should be able to find the same folder in other source engine games,changing the folder names)

any .cfg file located in that folder may be execured using the exec <Filename.cfg>

the way i implemented this method (in node.js) is to:

1) write the wanted console command to a chaos.cfg file in the cfg directory
2) sleepfor a few ms to allow for the file write to complete
3) use robotjs (http://robotjs.io/docs/)to press the ']' key (the ']'key because noone uses it while playing portal)
4) the game will receive the keypress, look for the .cfg file, see it hasthe command we want to execute in it, then run that commandin the console
as if someone had executed it directly

5) (Optional) if the effect is a temporary effect, such as low fps, i use the 'SetTimeout' function in javascript to run the reversing command after the effect ends.

pros: 
    pretty fast, uses ingame features fully documented to function

cons:
    if the user isn't tabbed into the game, the effect will not take place.
    if the user isn't tabbed into the game when a temporary effect ends, the effect will last indefinitely (unless the effect is chosen again, and the user is tabbed in after that one ends)
    high chance of effects failing.
    (as it's implemented in this version of the chaos mod) node.js used, therefore GUI is not a viable choice.
    restricted to only console commands.
    general jankiness



2: the netconsole method

this is the middle ground of all the methods
this requires for portal 2 to be launched with the "-netconport <port>" launch argument via steam, meaning the 1st time installation 
is more complicated.

this method uses the source engine's net console feature, essentially allowing for clients to connect to the source console via TelNet.
this means they have full access to console using networking.

to get setup:

1) launch portal 2 with the argument "-netconport <port>" argument, to do so, find portal 2 on steam, goto Settings > Properties > Launch options
and adding the argument (replace <port> with a random number, avoiding commonly used ports such as https://www.hostpapa.co.uk/knowledgebase/commonly-used-ports/)

2) enable TelNet for windows (this is only to let you connect using the windows commandprompt) by going to "Turn windows features on and off", finding "TelNet client"
and enabling in windows.

3) open the commandprompt, then run the command "telnet localhost <port>", it should now load into the console, allowing you to run commands.

pros:
    valve authorised feature
    doesn't require for keypresses to be used, meaning the user can me tabbed out while an effect runs.

cons:
    more in-depth installation for the end user
    limited to only console commands and limits for convars
    ports may overlap with other apps running on the client's PC

<End alternate method section>



Getting onto the real implementation of portal 2 chaos: the Direct memory hacking method.

this documentation should go through basic game hacking ideas and implementations, such as offsets, how to find them, and how to use them, 
accessing memory of the game, and hooking into the game externally.

a majority of this code will be ripped from my previous adventure into the cs:go cheat development scene.

firstly, we need to hook into the game, for this example, I will be making this in a c++ Console app.

starting with a simple :

#include <iostream>        <-- allows for console in/out
#include <Windows.h>       <-- allows for us to find and hook the portal window
#include <TlHelp32.h>      <-- general memory and stuff idk just put it however

int main()
{

}

at first, we need to get the 'HWND' of Portal 2, a 'HWND' stands for the 'Handle to the window', allowing usto access the window.

to do this, we run a simple 'FindWindowA()' command.
FindWindowA requires a 'class name'and a 'window name', the class name we can leave as null (NULL in c++),and for the window name, "PORTAL 2 - Direct3D 9"
the window name may change based on game settings ect, so make sure to equate for that in your code.
FindWindowA returns the 'HWND' of the window, we can make this into a global variable for the restof the methods to use.

the next thing we need is the process ID, that we can get with the function 'GetWindowThreadProcessId'
GetWindowThreadProcessId requires a HWNDof a window, of which, we can just use the HWND we received from the FindWindowA above,
GetWindowThreadProcessId also needs an output variable, so we can create a global 'DWORD' named procid, so for the 2nd argument, you can enter &procId, meaning the output
of this function will be pushed into the procId variable.

we now need to find the 'modulebase' of the client, so we need to access 'client.dll' in portal 2,to get which, use the below c++ code or equivalent:

uintptr_t GetModuleBaseAddress(const char* modName) {
	HANDLE hSnap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, procId);
	if (hSnap != INVALID_HANDLE_VALUE) {
		MODULEENTRY32 modEntry;
		modEntry.dwSize = sizeof(modEntry);
		if (Module32First(hSnap, &modEntry)) {
			do {
				if (!strcmp(modEntry.szModule, modName)) {
					CloseHandle(hSnap);
					return (uintptr_t)modEntry.modBaseAddr;
				}
			} while (Module32Next(hSnap, &modEntry));
		}
	}
}

then make a global uintptr_t called module base, which is assigned the value of "GetModuleBaseAddress("client.dll")"

this allows us to access variables within the client.dll, including all of the console variables.

next we need to actually open and hook into portal 2, by using 'OpenProcess', the argumentsbeing 'PROCESS_ALL_ACCESS', 'NULL', then our procId

assign this to a global HANDLE variable.

from here on, we can use ReadProcessMemory, and WriteProcessMemory to read and write to memory within the game, such as the players' FOV, fps_max, and any other console variable.

from this method, there is no need for any other installation steps by the user, as even things such as sv_cheats 1 can be edited automatically.
However, to edit console variables, and other values, you need what is called an 'offset'
and offset is used to show the location in memory where we can read/write to find the specific variable.

as an example, the offset for the FOV variable is: 0xA04C54
therefore, the location in memory would be: (modulebase + 0xA04v54).

to access this in c++, we can use the ReadProcessMemory function, 
we pass to ReadProcessMemory :
hProcess      : so it knows which process to look for the variable in
address       : this is the '(modulebase + offset)' value
buffer        : this is the output float for the result to be written to
sizeof(float) : so it knows how far into memory to read without it reading other binary data
NULL          : fill in unused arguments.

this should output the value of the FOV (r other offset's) console variable/value.


to write to the memory, to edit console variables and other values, you must use 'WriteProcessMemory'

WriteProcessMemory is almost the same as ReadProcessMemory, however instead of the buffer being written to, the buffer is being read from,
with the 'buffer' including the new wanted value of the value being written.

as an example of both of these functions, this is a function that reads current FOV, then sets fov to +1, increasing the FOV by 1:

void increaseFovByone(){
    float previousFOV    //the 'buffer' input
	ReadProcessMemory(hProcess, (LPCVOID)(moduleBase + convar_fov), &buffer, sizeof(float), NULL);

    float newFOV = previousFOV + 1;
	WriteProcessMemory(hProcess, (void*)(moduleBase + convar_fov), &newFOV, sizeof(float), NULL);
}

to add additional console variables, you need to find the offset of the variable.

this is not a cheat engine tutorial, however you will only need a basic knowledge of cheat engine:

i would recommend watching this Guided Hacking video: https://www.youtube.com/watch?v=Nib69uZJCaA&ab_channel=GuidedHacking

after you have the variable in your cheat table, you need to acquire the offset. cheat engine supplies this under the 'Address' column on the cheat table.

to translate this offset into an offset you can use in your code: you need to do a few things:

you need to know whether or not the variable is within the client.dll, you can find this out by clicking on the 'address' variable on your cheat table entry,
if the 'address' field in the popup window has a 'cliend.dll + ' in it, your variable is within the client.dll

you then need to copy this address, excluding the client.dll.you can then import then in you your source c++ file in either an included header file or the cpp file,
by using #define.

for example: 
#define convar_fov 0xA04C54

#define <name> <offset>

to use this in Read/WriteProcessMemory, you will need to find the actual address,
as you figured out above, you should know if your variable is within client.dll, you will need to add modulebase + offset, iff not, you can leave it as just offset.

you can now find, read, and write console variables completely externally.

i'm working on documenting and dumping all of the offsets for portal 2 and uploading them for people to see on my github: https://github.com/Taob-arch

if you want to help, you can try and find any offset i haven't found and submit it to my github :)
