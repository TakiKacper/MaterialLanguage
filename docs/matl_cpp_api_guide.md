# Matl Api
Integrating matl with Your engine is a pretty easy task, thanks to it's simple api.

## Contents
- [Building](#Building)
- [Minimal matl integration step by step](#Minimal-matl-integration-step-by-step)
  - [Creating context](#Creating-context)
  - [Parsing domains](#Parsing-domains)
  - [Parsing materials](#Parsing-materials)
  - [Destroying context](#Destroying-context)
  - [Minimal example](#Minimal-example)
- [Implementing rest of matl features](#Implementing-rest-of-matl-features)
  - [Parsing libraries](#Parsing-libraries)
  - [Using domain insertions](#Using-domain-insertions)
  - [Commonly exposed functions](#Commonly-exposed-functions)
  - [Custom using cases](#Custom-using-cases)

## Building  
Building matl is similar to compiling single-header library, except you must include several files: matl parser (matl.hpp) and translators.

1. Clone this repo
2. Move source and ``matl.hpp`` files into your project files
3. Create .cpp file ``matl_impl.cpp``
4. Inside put the line ``#define MATL_IMPLEMENTATION``
5. Include ``matl.hpp`` and all of the translators
6. Compile
   
## Minimal matl integration step by step
### Creating context
First of all, we must create a context. Context is an object that contains all the informations required to parse matl material into a working shader.
```cpp
#include "(path)/matl.hpp"

...
matl::context* context = matl::create_context("opengl_glsl");
```
The argument of create_context is targeted shader language. The function will fail and return nullptr if there is no translator available for such language.

### Parsing domains
As mentioned in readme, domain is a template of a shader. See ``docs/matl_domains_programming_guide.md`` for more information about them. 
```cpp
matl::domain_parsing_raport raport = matl::parse_domain("my_domain", domain_source, context);
for (auto& error : raport.errors) std::cout << error << '\n';
if (!raport.success) handle_error();
```
matl::parse_domain is a function that loads domain and saves it inside the context. It also returns struct with informations about operation's successfulness and errors.
<details>
  <summary>domain_parsing_raport definition</summary>

```cpp
struct domain_parsing_raport
{
    //Whether parsing was successful and there are no errors
    bool success = false;

    //Parsing errors
    std::list<std::string> errors;
};
```
  
</details>
  
First argument is the domain name - this name will be used inside materials, in ``using domain (domain name)`` line.  
In this simple example we use name ``"my_domain"`` but it is suggested that you use something that represents used shading model or domain use case eg. surface-lit, postprocess, ui etc.   
  
The ``domain_source`` must be of type std::string.

It is worth noting that parser fails to parse the domain, it will not be saved inside the context, making it not available for materials.  
  
You can repeat this step multiple times to add multiple domains.

### Parsing materials
Once you have at least one domain loaded into your context you can attempt translating materials.
```cpp
matl::parsed_material mat = matl::parse_material(material_source, context);
for (auto& error : mat.errors) std::cout << error << '\n';
if (!mat.success) handle_error();

...

//If parsing was successfull you can use generated shader sources to create a shader on the gpu
auto shader = create_shader(mat.sources);
```
matl::parse_material takes matl material as a first arg and returns a parsed_material struct.

<details>
<summary>parsed_material definition</summary>

```cpp
struct parsed_material
{
    //Whether parsing was successful and there are no errors
    bool success = false;

    //Shader code in target language
    std::list<std::string> sources;

    //Parsing errors
    std::list<std::string> errors;

    struct parameter
    {
        std::string name;

        enum class type : uint8_t
        {
            boolean,
            scalar,
            vector2,
            vector3,
            vector4,
            texture
        } type;

        std::list<float> numeric_default_value;
        std::string		 texture_default_value;
    };

    //Parameters (directx constants, opengl uniforms ...) generated by material
    std::list<parameter> parameters;
};
```

</details>

### Destroying context
After you parse all the materials, you should free the context memory using
```cpp
matl::destroy_context(context);
```

### Minimal example
At this point your application may look like this:
  
<details>
  <summary>Example</summary>

matl_impl.cpp  
```cpp
#define MATL_IMPLEMENTATION
#include "include/matl/matl.hpp"
#include "include/matl/matl_glsl.hpp"
```

main.cpp  
```cpp
#include <iostream>
#include <fstream>

#include "include/matl/matl.hpp"

std::string get_file(const std::string& file_name)
{
    std::fstream t(file_name);

    t.seekg(0, std::ios::end);
    size_t size = t.tellg();
    auto source = std::string(size, ' ');
    t.seekg(0);
    t.read(&source[0], size);

    t.close();

    return source;
}

std::string save_to_file(std::string filename, std::list<std::string>& sources)
{
    std::ofstream content;
    content.open(filename);
    for (auto& source : sources)
	content << source;

    content.close();
}

int main()
{
    //Create context
    auto context = matl::create_context("opengl_glsl");

    //Parse Domain
    matl::domain_parsing_raport dpr = matl::parse_domain("my_domain", get_file("domain.glsl"), context);

    //Parse Material
    matl::parsed_material pm = matl::parse_material(get_file("material.matl"), context);

    //Print Errors
    std::cout << "Domain Errors\n";
    for (auto& err : dpr.errors)
	std::cout << err << '\n';

    std::cout << "Material Errors\n";
    for (auto& error : pm.errors)
	std::cout << error << "\n";

    save_to_file("result_shader.glsl", pm.sources);

    matl::destroy_context(context);
}
```
</details>


## Implementing rest of matl features
### Parsing libraries
Libraries are matl files that simply contains matl functions.
<details>
<summary>Example library: </summary>
	
 ```rust
func lerp(y0, y1, alpha)
	return y0 + (y1 - y0) * alpha

func snap_to_grid(point, grid_size)
	return floor(point / grid_size) * grid_size
```

</details>

Once library is parsed, it is saved inside ``context`` and can be used by every material via ``using library (name)``.  
Parsing simple library as one the above is pretty straightforward:

```cpp
std::list<matl::library_parsing_raport> lprs = matl::parse_library("my_library", library_source, context);
```
But this only work for simple libraries that does not use other libraries.   
With more complex ones we will eventually run into a problem, when library wants to use another library that is not already loaded.
 ```rust
using library lerp_impl	# lerp_impl might not be in the context yet!

func lerp(y0, y1, alpha)
	return lerp_impl.lerp(y0, y1, alpha)
```
The soultion is to provide a context callback function that will allow matl parser to request source of missing library from the client.
```cpp
std::list<std::string> buffer;
const std::string* lib_source_request(const std::string& name, std::string& error)
{
	buffer.push_back(get_file(name + ".matl"));
	return &buffer.back();
}

//Need to be set only once
context->set_library_source_request_callback(lib_source_request);

(...)

matl::library_parsing_raport lpr = matl::parse_library("my_library", library_source, context);
```
Now, when parser does not have required library it will just call to ``lib_source_request`` for it.
The callback functions takes two arguments:
1. Requested library name - in above example it would be lerp_impl
2. Error - an error message - use if your application cannot find or access given library source

Pay attention that Matl parser does not take the ownership of the memory - you must store and release it on your own.

Return value of ``matl::parse_library`` is diffrent from other parse functions - it returns a list of raports instead of a single one. 
This is because of fact that due to ability to request libraries sources, ``parse_library`` might actually parse more than one lib at one call.

```cpp
std::list<matl::library_parsing_raport>
```
<details>
	<summary>library parsing raport definition: </summary>

```cpp
struct library_parsing_raport
{
	//Whether parsing was successful and there are no errors
	bool success = false;

	//Parsed library name
	std::string library_name;

	//Parsing errors
	std::list<std::string> errors;
};
```
</details>

### Using domain insertions
Insertions are code snippets that can be pasted into the shader code during it's assembly.  
They can be saved inside context using following code:  
```cpp
context->add_domain_insertion("vertex_layout", "layout (location = 0) in vec2 aPos; [...]")
```
The first arg is insertion name and the second is it's content.  
Once saved insertion can be pasted into the shader using following domain directive:
```
<dump insertion vertex_layout>
```
### Commonly exposed functions
Languages like glsl or hlsl comes with a set of builtin functions like lerp or smoothstep. 
Using those prebuilit ones, instead of own implementations of those functions will always be faster, because those will be accelerated and optimized to fully take advantage of the given hardware, 
therefore it would be nice to make them available, to the materials.
As You maybe know it can be done from domain using the ``function`` direcive from within the ``expose`` block, but doing that in every domain would be a huge repetition.  
To solve this problem use the ``context::add_commonly_exposed_functions`` method. It takes only one argument: a source of domain that contains only ``<expose>``, ``<end>`` and ``function`` directives.

<details>
	<summary>example</summary>
	
```glsl
<expose>
    <function   lerp = scalar  lerp(scalar, scalar, scalar)>
    <function   lerp = vector2 lerp(vector2, vector2, vector2)>
    <function   lerp = vector3 lerp(vector3, vector3, vector3)>
    <function   lerp = vector4 lerp(vector4, vector4, vector4)>

    <function   lerp = vector2 lerp(vector2, vector2, scalar)>
    <function   lerp = vector3 lerp(vector3, vector3, scalar)>
    <function   lerp = vector4 lerp(vector4, vector4, scalar)>
<end>
```
	
</details>
Functions exposed by this domain will now be available to every material no matter the domain.  

The ``context::add_commonly_exposed_functions`` returns a ``add_commonly_exposed_functions_raport`` struct with information about the operation success.

<details>
	<summary>add_commonly_exposed_functions_raport definition</summary>

```cpp
struct add_commonly_exposed_functions_raport
{
	//Whether functions were successfully added
	bool success = false;

	//Parsing errors
	std::list<std::string> errors;
};
```
  
</details>

### Custom using cases
Matl allows user to add their own ``using`` keyword overload. This can be done from c++ by using ``context::add_custom_using_case_callback`` method:
```cpp
void using_print(std::string args, std::string& error)
{
	std::cout << "PRINT: \"" + args << "\"\n";
}

context->add_custom_using_case_callback("print", using_print);
```
The first arg of ``add_custom_using_case_callback`` is the word that should result in a callback.
The second arg is of course the callback function, which takes rest of the line as the args argument, and reference to the error string that can be used to pass an error message.  

With above setup, when matl parser come across
```
using print a b cde fg
```
It will call to ``using_print`` with ``args = "a b cde fg"``.













