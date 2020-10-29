---
title: Using Bison C++ API With Hand-written Scanner
date: 2020-10-24 22:00:00 +/-0800
categories: [Compilers, Bison]
tags: [Cpp, Bison]
---
In this post, I'll talk about how can you use Bison's C++ API. Take a look at the [tutorial I wrote about using C API](https://thesharpowl.github.io/posts/HAND_WRITTEN_SCANNER_WITH_BISON_C_API/) especially the parts about installing Bison latest version and Compiler Construction. Also, to gain more knowledge about how Bison works.<br>
<br>
1. Well, as I said in the C API post, you can't use dynamic types such as ``std::string`` or ``std::vector`` as Bison types which is one of the reasons of using C++ API (The main reason is that C++ is cooler of course :sunglasses: ). 

    Again this tutorial is not focusing on Bison itself, just the C++ API part. 

2. We have a simple input file in an imperative language :<br>
```
    var some_identifer is integer
```
<br>
3. Now let's write a simple Scanner in C++ (same idea as the C one) :<br>

```
#include<iostream>
#include<fstream>
#include<string>

// I know that global variables are often bad. Forgive me I wanna just explain the idea ((
std::ifstream fin;

std::string get_next_token()
{
    std::string s;
    char c;
	
    while (true)
    {
		// could be done better but organized it like this to edit only assignments and returns when using Bison API
        if (!fin.eof())
            fin.get(c);// get one character
    	
        if (c == ' ' || c == '\n' || fin.eof())
        {
            if (s.empty()) // we only have this character
            {
                if (fin.eof())
                    return "";
                //otherwise go and see what's next
                return get_next_token();
            }
            else
            {
                // now we need to put the pointer exactly after the word (undo the last step)
                fin.unget();
            	
                if (s == "var") // the last word is var
                    return s;
                if (s == "integer")
                    return s;
                if (s == "is")
					return s;
                if (!s.empty())// it means it is some identifier name
                    return  "Identifier";
            }
        }
        else if ((c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z') || c == '_') // reading some name
        {
            s += c; // add the char to the string
        }
        else
        {
        	// we don't know what's that
            return "ERROR";
        }
    }
}
int main()
{
    
    fin.open("input.txt");
    std::string temp = get_next_token();
	while(!temp.empty())
	{
		// printing a space since our example is only one line for now
        std::cout << temp << " ";
        temp = get_next_token();
	}
    return 0;
}
```

4. Now the Bison part. (not going to go deep into grammar rules or the way you design them. I learned grammar in Theoretical Computer Science course) but I will talk about the programming aspects.<br>

    * Bison file consists of 4 parts :

        I. Defines and requirements (statementsstart    with ``%`` usually).

        II. C code part(s) for includes andsignatures   (will be at the beginning of thegenerated .h  file)

        III. Grammar part

        IV. Function defintions part

5. Take a look at this Bison with C++ example (I will explain after that) :

```
%require "3.2"
%language "C++"
%define api.value.type variant
%define api.token.constructor
%define parse.assert

%code requires
{
    #pragma once
    #include <iostream>
    #include <string>
    
    // forward decleration (1)
    namespace yy
    {
        class parser;
    }
}

%code
{    
    namespace yy
    {
        parser::symbol_type yylex(); // override (2)
        // no need to override the error function because it is already waiting for our implementation (3)
    }
}

%token IDENT VAR INT IS
%type <std::string> IDENT

%%

VariableDeclaration:
|VAR IDENT IS INT { /* no actions for now (4) */ }
;

%%
namespace yy
{
    parser::symbol_type yylex()
    {
        return  yy::parser::make_YYEOF ();
    }
    void parser::error(const std::string& msg) //(3+)
    {
        std::cout<< "syntax error!\n";
    }
}

int main()
{
    yy::parser p; // (5)
    return 0;
}
```

(Warn about ypp extension !!!!!!!!!!!!!!)
To Be continued......