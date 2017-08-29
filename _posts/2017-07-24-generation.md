---
layout: post
title: My 2017 GSoC Project - Part IX
subtitle: Translators Needed
category: Programming
tags: [GSoC-2017, Scilab, Modelica]
---

Hello again,

In a much needed attempt to deliver more meaningful results before **GSoC's Second Evaluation**, today I'm applying information from [the previous post]({% post_url 2017-07-16-description %}) on actual generation of a **Modelica**-based **Scicos** block, using **OpenModelica**.

For reasons beyond my knowledge (no sarcasm intended), **Scilab** currently uses [**OCaml**](https://en.wikipedia.org/wiki/OCaml) language for writing the executables involved in **Modelica** simulation. And, as they seem to be the only pieces that need a **OCaml** compiler available to be built, being able to replace them completely by **OMCompiler** looks like a nice dependency tradeoff.

### Modelica Code Generation

The first step in **Modelica** simulation (at least after construction of the diagram structure) is generation of code. During the first compilation pass (at **c_pass1.sci** script), all found **Modelica** blocks are added to a list to be combined into a single one by **build_modelica_block.sci** function:

{% highlight javascript %}
function [model,ok] = build_modelica_block( blklstm, corinvm, cmmat, NiM, NoM, NvM, scs_m, path )
    // Given the blocks definitions in blklstm and connections in cmmat this
    // function first create  the associated modelicablock  and writes its code
    // in the file named 'imppart_'+name+'.mo' in the directory given by path
    // Then modelica compiler is called to produce the C code of scicos block
    // associated to this modelica block. filbally the C code is compiled and
    // dynamically linked with Scilab.
    // The correspondind model data structure is returned.


    // Get the name of the generated main modelica file
    name = stripblanks( scs_m.props.title(1) ) + "_im";

    // Generation of the txt for the main modelica file
    // plus return ipar/rpar for the model of THE modelica block
    [txt, rpar, ipar] = create_modelica( blklstm, corinvm, cmmat, NvM, name, scs_m );

    // Write txt in the file path+name+'.mo'
    path = pathconvert( stripblanks( path ), %t, %t );
    mputl( txt, path + name + ".mo" );

    // Append data from all Modelica blocks found to list
    Mblocks = [];
    for i=1:lstsize( blklstm )
        if type( blklstm(i).sim ) == 15 then
            if blklstm(i).sim(2) == 30004 then
                o = scs_m( scs_full_path( corinvm(i) ) );
                Mblocks=[ Mblocks;
                          o.graphics.exprs.nameF ];
            end
        end
    end

    // Generate XML and model_flat_Model
    // Compile modelica files to binary library
    [ok,name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = compile_modelica( path + name + ".mo", Mblocks );

    if ~ok then return,end

    // Build model data structure of the block equivalent to the implicit part
    model = scicos_model( sim=list( name, 10004 ),..
    label=name, uid=guid,..
    in=ones( nin, 1 ), out=ones( nout, 1 ),..
    state=zeros( nx*2, 1 ),..
    dstate=zeros( nz, 1 ),..
    rpar=rpar,..
    ipar=ipar,..
    dep_ut=[ dep_u %t ],nzcross=ng,nmode=nm );
endfunction
{% endhighlight %}

Inside **create_modelica** functions (defined at **create_modelica.sci**), information from all **Modelica** blocks is combined (considering the **interconnection matrices** passed as arguments) and a single **.mo** source file is generated. All this process is performed in **Scilab** script and is independent of our implementation, meaning that it remains untouched.

### Model Flattening

By default, a **Scilab** installation already contains **Modelica** libraries of **Mechanical** and **Electrical** components (that could be expanded by **toolboxes** like [**Coselica**](https://atoms.scilab.org/toolboxes/coselica)) to be used by more complex models. After generation of the **superblock** code, passed to **compile_modelica** function, the relevant (utilized) parts of those libraries are also added to the main file by **translator** function, creating the final **flat** model:

{% highlight javascript %}
function [ok,name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = compile_modelica( filemo, Mblocks )

    // ...

    // Verify if simulation uses analytical Jacobian matrix
    if exists("%Jacobian") == 0 then %Jacobian=%t; end

    //Initialize lhs arguments in case of return on error
    name=""; guid="";                           // Model name and GUID
    dep_u=%t; nipar=0; nrpar=0; nopar=0;        // Input/output dependency and number of parameters
    nz=0; nx=0; nx_der=0; nx_ns=0;              // Number of states and derivatives
    nin=0; nout=0;                              // Number of inputs and outputs
    nm=0; ng=0;                                 // Number of modes and zero-crossing surfaces

    //set paths for generated files
    outpath = pathconvert( TMPDIR, %t, %t );

    model_name = basename( filemo );
    model_name_flat = model_name + "f";

    // Flat modelica file generated by translator
    model_flat_file = outpath + model_name_flat + ".mo";

    // Files/folders generated by modelica compiler
    model_desc_file = outpath + model_name + "/modelDescription.xml"; // Model description XML file
    model_C_file = outpath + model_name + ".c"                        // C code file generated for flat model

    // ...

    // Generate with filemo and Modelica libraries the flat model
    ok = translator( filemo, Mblocks, model_flat_file );
    if ~ok then  dep_u=%t; return,end

    // ...

{% endhighlight %}

Inside **translator**, the **OCaml**-generated **modelicat** executable is replaced by **OMCompiler**, which offers the same flattening capabilities:

{% highlight javascript %}
function ok = translator( filemo, Mblocks, Flat )
    //Generate the flat model of the Modelica model given in the filemo file
    //and the Modelica libraries. Interface to the external tool
    //translator.

    // Get OMcompiler executable name
    TRANSLATOR_FILENAME = "omc";
    if getos() == "Windows" then
        TRANSLATOR_FILENAME = TRANSLATOR_FILENAME + ".exe";
    end

    // Get libraries and output paths
    [ modelica_libs, modelica_directory ] = getModelicaPath();

    // Find all needed libraries
    // ...

    translator_libs = " """ + filemo + """ " + translator_libs;

    //Build the shell instruction for calling the translator

    exe = getmodelicacpath() + TRANSLATOR_FILENAME
    exe = """" + pathconvert(getmodelicacpath() + TRANSLATOR_FILENAME,%f,%t) + """ ";

    out =" """ + Flat + """" // Flat modelica

    // Shell instruction for generating flat Modelica code and saving it to a file
    // --modelicaOutput sets output to be a modelica source file
    // --useLocalDirection preserves the input/output properties from library variables
    // --reduceTerms remove input/output redundancies
    instr = exe + " " + translator_libs + " --modelicaOutput --useLocalDirection --reduceTerms > " + out;

    // ...

    // Instruction call and error handling
    [rep,stat,err]=unix_g(instr);
    if stat <> 0 then
        messagebox(err, _("Modelica translator"), "error", "modal");
        ok=%f;
        return
    end

    ok = %t

endfunction
{% endhighlight %}

### C code generation

Next, **FMU package** and **FMI2 C wrapper** code are generated from **flat** model:

{% highlight javascript %}
function [ok,name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = compile_modelica( filemo, Mblocks )

    // ...

    //Generate the C file with modelicac
    ok = modelicac( model_flat_file, %Jacobian, running=="1", model_C_file, model_desc_file );
    if ~ok then return,end

    // Read XML file data into a tree-like native data structure
    [name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = reading_incidence( model_desc_file );

    // ...

    //compile and link the generated C file
    ok = Link_modelica_C( model_C_file );

{% endhighlight %}

Inside **modelicac** function (the name was kept the same as in previous implementation), **OMCompiler** is once again used, running the **FMU package** creation and extraction [**.mos script**](https://build.openmodelica.org/Documentation/OpenModelica.Scripting.html). Also, our **C** code, already explained in past publications, is used on this step:

{% highlight javascript %}
function  ok = modelicac( model_flat_file, Jacobian, model_C_file, model_desc_file )
    //Scilab interface with external tool modelicac

    MODELICAC_FILENAME = "omc";
    if getos() == "Windows" then
        MODELICAC_FILENAME = MODELICAC_FILENAME + ".exe";
    end

    [model_path, model_name, C_ext] = fileparts(model_C_file);
    [model_path, model_flat_name, flat_ext] = fileparts(model_flat_file);

    model_flat_script = pathconvert(model_path + model_name + ".mos", %f, %t); mprintf("modelicac: script file: %s\n", model_flat_script);
    model_flat_package = pathconvert(model_path + model_name + ".fmu", %f, %t); mprintf("modelicac: packege file: %s\n", model_flat_package);

    current_dir = pwd();
    chdir( model_path );

    if getos() == "Windows" then
        extract_cmd = getshortpathname(pathconvert(SCI+"/tools/zip/unzip.exe",%F)) + " -o ";
    else
        extract_cmd = "unzip -o ";
    end

    // Generate OpenModelica script file (builds FMU package with partial derivative support and unzips it)
    compile_commands = [];
    compile_commands($ + 1) = "setModelicaPath(""" + model_path + """); getErrorString();"
    // Try to use Visual Studio's native compiler on Windows systems
    if getos() == "Windows" then
        compile_commands($ + 1) = "setCommandLineOptions(""+target=msvc""); getErrorString();";
    end
    if Jacobian then
        compile_commands($ + 1) = "setDebugFlags(""fmuExperimental""); getErrorString();";
    end
    compile_commands($ + 1) = "loadFile(""" + model_flat_name + flat_ext + """); getErrorString();";
    compile_commands($ + 1) = "translateModelFMU(" + model_name + "); getErrorString();";
    compile_commands($ + 1) = "system(""" + extract_cmd + model_flat_package + """); getErrorString();";
    mputl( compile_commands, model_flat_script );

    exe = """" + pathconvert(getmodelicacpath() + MODELICAC_FILENAME, %f, %t) + """";
    mprintf("modelicac: executable: %s\n", MODELICAC_FILENAME);
    model_flat_script = """" + model_flat_script + """";

    instr = strcat([exe, model_flat_script], " ");

    // ...

    [rep,stat,err]=unix_g(instr);

    chdir( current_dir );

    if stat <> 0 then
        messagebox(err, _("Modelica compiler"), "error", "modal");
        ok=%f;
        return
    end

    //Modelica library C code wrapper for the simulation function
    fmi2_wrapper_code = generate_fmi2_wrapper(model_desc_file);
    mputl(fmi2_wrapper_code, model_C_file);

endfunction

function code = generate_fmi2_wrapper(model_desc_file)
    // Get variable references lists from description file
    model_desc_tree = xmlRead(model_desc_file);
    [name,guid,in_refs,x_refs,x_der_refs,par_refs,out_refs]=read_modelica_variables(model_desc_tree);

    in_length = max(1, length(in_refs));
    x_length = max(1, length(x_refs));
    x_der_length = max(1, length(x_der_refs));
    out_length = max(1, length(out_refs));
    par_length = max(1, length(par_refs));

    // Generate C code for specific fixed size variables lists and append generic FMI2 wrapper code
    code = [
               "#include ""fmi2Functions.h""",
               "#include ""fmi2FunctionTypes.h""",
               "#include ""fmi2TypesPlatform.h""",
               "",
               strcat(["static fmi2Real inputsList[ ", string(in_length), " ] = { 0.0 };"]),
               strcat(["static const fmi2ValueReference INPUT_REFS_LIST[ ", string(in_length), " ] = { ", strcat(string(in_refs), ","), " };"]),
               strcat(["static const fmi2ValueReference STATE_REFS_LIST[ ", string(x_length), " ] = { ", strcat(string(x_refs), ","), " };"]),
               strcat(["static fmi2Real stateDerivativesList[ ", string(x_der_length), " ] = { 0.0 };"]),
               strcat(["static const fmi2ValueReference STATE_DER_REFS_LIST[ ", string(x_der_length), " ] = { ", strcat(string(x_der_refs), ","), " };"]),
               strcat(["static fmi2Real outputsList[ ", string(out_length), " ] = { 0.0 };"]),
               strcat(["static const fmi2ValueReference OUTPUT_REFS_LIST[ ", string(out_length), " ] = { ", strcat(string(out_refs), ","), " };"]),
               strcat(["static fmi2Real parametersList[ ", string(par_length), " ] = { 0.0 };"]),
               strcat(["static const fmi2ValueReference PARAMETER_REFS_LIST[ ", string(par_length), " ] = { ", strcat(string(par_refs), ","), " };"]),
               "",
               strcat(["#define BLOCK_FUNCTION_NAME ", name]),
               strcat(["#define MODEL_NAME """, name, """"]),
               strcat(["#define MODEL_GUID """, guid, """"]),
               "",
               "#include ""fmi2_wrapper.h"""
           ];

endfunction
{% endhighlight %}

Notice that our **FMI2** layer code is put into a header (**.h**) file now (located in **/system_includes_dir/scilab/**), as binary distributions of **Scilab** doesn't ship sources (**.c**). Also, as we have to generate the actual **C** source file including the wrapper, this gives the opportunity to add the **variable references** (indexing inputs, outputs, states, etc...) vectors that couldn't be passed to the **computational function** otherwise.

### Model Description Parsing

After that, **modelDescription.xml** file produced by **OMCompiler** is parsed in order to extract all properties needed for the **Scicos** block:

{% highlight javascript %}
function [ok,name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = compile_modelica( filemo, Mblocks )

    // ...

    // Read XML file data into a tree-like native data structure
    model_desc_tree = xmlRead( model_desc_file );

    [name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = reading_incidence( model_desc_file )

    // ...

{% endhighlight %}

{% highlight javascript %}
function [name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = reading_incidence( model_desc_file )
    // this function creates the matrix dep_u given by the xml format.
    // It is used for modelica compiler.
    // number of lines represents the number of input, number of columns represents the number of outputs.

    model_desc_tree = xmlRead( model_desc_file );
    [name,guid,in_refs,x_refs,x_der_refs,par_refs,out_refs] = read_modelica_variables( model_desc_tree );

    nipar = 0; nopar = 0; nz = 0; nx_ns = 0; nm = 0;
    nin = length(in_refs);
    nx = length(x_refs);
    nx_der = length(x_der_refs);
    nout = length(out_refs);
    nrpar = length(par_refs);

    // Aquire and display number of model's event indicators
    ng = round( strtod( model_desc_tree.root.attributes( "numberOfEventIndicators" ) ) );

    // Output includes state, so is it always depends on existing input
    if nin > 0 then
        dep_u = %t
    else
        dep_u = %f
    end

endfunction
{% endhighlight %}

Here, **read_model_variables** works much like in [the previous post]({% post_url 2017-07-16-description %}).

### Model Library Linkage

Finally, system's native compiler is called for building the **wrapped FMI2 library** is link it to simulator main executable:

{% highlight javascript %}
function [ok,name,guid,nipar,nrpar,nopar,nz,nx,nx_der,nx_ns,nin,nout,nm,ng,dep_u] = compile_modelica( filemo, Mblocks )

    // ...

    //compile and link the generated C file
    ok = Link_modelica_C( model_C_file );

endfunction
{% endhighlight %}

{% highlight javascript %}
function ok = Link_modelica_C( model_C_file, model_path )

    model_C_file = pathconvert(model_C_file, %f, %t)
    [model_path, model_name, ext] = fileparts(model_C_file);

    // Determine binary libraries directory according to platform

    [version, opts] = getversion();
    compiler = opts(1)
    if opts(2) == "x64" then
        arch = "64"
    else
        arch = "32"
    end

    model_libs_path = model_path + "/" + model_name + "/binaries/";
    if getos() == "Windows" then
        model_libs_path = model_libs_path + "/win" + arch + "/";
    else
        model_libs_path = model_libs_path + "/linux" + arch + "/";
    end

    // Define FMI2 headers include directory
    model_include_path = model_path + "/" + model_name + "/sources/include/fmi2/";

    // Add linked libraries list
    model_libs = [ model_libs_path + model_name + getdynlibext() ];

    // ...

    // Define list of directories to be searched for *.h
    Cpp_flags = "  -I""" + model_include_path + """";

    // Call common function for building block shared library from C code
    ok = buildnewblock(model_name, model_name, "", "", model_libs, TMPDIR, "", IncludePaths);

endfunction
{% endhighlight %}

### Final Considerations

The implementation showed here still has rough edges, as it is troublesome to understand the purpose of all variables and operations involved in the original codebase, and it's not properly tested. Even so, it constitutes the bulk of the work involved in interfacing **Scilab** with **OpenModelica's compiler**. Over time, this post will be updated with eventual refinements applied to the code.

That's it for now. Thanks one more time for sticking by. See you soon !
