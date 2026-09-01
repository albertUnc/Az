# README.txt
I'm using a .txt instead of a .md because i don't know how to write MDs and it annoyed me when i tried
This file only contains little stuff, no details at all. Check language-info.txt for more details,
inside it there will be path guides and links to documentation, tutorials.

This is Az. A language I made in C++.
I used a lot of AI in this, but i do actually know how to code in C++, at a level where
i could've coded it by hand, but it's obviously much slower and harder.

How does the language work?
The .az source files get transpiled into real C++ by az-transpiler.exe
(which was by the way also made in c++), then you just use a normal c++
compiler like g++ on the transpiled file.
With multiple files:
in Az, you always only trasnpile 1 file, that file CAN include others with
the keyword using, it's a mix of python and c++, but more similar to python
The transpiler checks for all functions inside the included additional .az files,
and allows the main file that's getting transpiled to use them. So it's basically
like a c++ .h file, other .az files can have functions. the main can also include
just one function from a file like this:
using MyOtherFile.az; //Whole file
using MyOtherFile.anyFunction; //Just 1 func

What are the features?
As more versions come out, this will change a lot.
Version 1 has basic stuff like variables, functions, methods,
built in libraries, loops, custom keywords, etc.
Version 2 will introduce OOP, deeper memory manipulating (pointers and references),
etc.

What about the syntax?
Here's a bit of code: (for example main.az):

using console;
using math;
//built-in libraries^

func sayName(): none //none is a keyword like None in python. After the : is the return type specified
{
    console.print(fstring("Your name is {name}")); //name can be used here since it was declared as global
}

main() //different function syntax for main, this starts by itself
{
    console.write("Yo\n");
    temp int value = console.input(int); //only allows int to get returned
    console.write(fstring("{value}^2 is {math.pow(value, 2)}\n"));
    //Times loop. Custom, introduced in this lang first
    console.write("Your number 3 times in a row:\n");
    times 3 {
        console.write(value); //converts to string automatically
    }
    tempdelete value;

    console.write("\n");
    console.write("Enter your name: ");
    const global string name = input(string);
    sayName();

    temp list<var> idk = ["Something", "Somethingelse"]; //var is a keyword for auto detecting the type
    temp dict<var, var> something = {"Unc": 'U', "Idk": 'I'};
    for var smth : idk { //C++ like iterating through an array
        console.write(smth);
        console.write(fstring("Current iteration index is: {_i}\n")); //_i is a variable accessible in for, while and times loops which holds the 0-based index
    }
    //also can use the init; condition; at the end of each it; loop:. Use none where you dont want to put anything
    for bool isDone = false; !isDone; none {
        int r = random.randint(1, 2); //chooses 1 or 2
        if r == 1 {isDone = true;}
        else {isDone = false;}
    }
}
//end of code

There are also more built in libraries including time, file I/O and more.
I made this language because i love C++, but some things are just hard to do and this has them built in, and also
I always wanted to make my own language.

Again, for more details, info for where documentation and tutorials are, go to language-info.txt

github: albertUnc 
(albert)
discord: albertunc
