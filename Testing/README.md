# Excel-DNA Testing Helper

_**Note: The Excel-DNA Testing Helper is in an early preview stage, and the design and functionality might change a lot in future. If you try it and have any feedback at all, positive or 'constructive' please let me know at govert@icon.co.za. If you already practice automated testing extensively I am particularly eager to hear of any suggestions you might have.**_

`ExcelDna.Testing` is a NuGet package and library that lets you develop automatic tests for Excel models and add-ins, including add-ins developed with Excel-DNA and VBA. Test code is written in C# or Visual Basic and is hosted by the popular [xUnit](https://xunit.net/) test framework, allowing automated tests to run from Visual Stuio or other standard test runners.

Tests developed with the testing helper will run with the add-in loaded inside an Excel instance, and so allow you to test the interaction between an add-in and a real instance of Excel. This type of 'integration testing' can augment 'unit testing' where individual library features are tested in isolation. It is often in the interaction with Excel where the problematic aspects of an add-in are revealed, and developing automated testing for this environment has been difficult.

The testing helper allows flexibility and power in designing automated Excel tests:
* The test code can either run in a separate process that drives Excel through the COM object model, or can be loaded inside the Excel process itself, allowing use of both the COM object model and the C API from the test code.
* Functions, macros and even ribbon commands can be tested.
* Test projects can include pre-populated test workbooks containing spreadsheet models to test or test data.

Running automated tests against Excel does introduce complications:
* Testing requires a copy of Excel to be installed on the machine where the tests are run, so don't work well as automated test for 'continuous integration' environments.
* Test outcomes can depend on the exact version of Excel the is used. This is both an advantage in identifying some 
* Integration tests with Excel can be quite slow to run compared to direct unit testing of functions.

This tutorial will introduce the Excel-DNA testing helper, and show you how to create a test project for your Excel model or add-in.

## Background and prerequisites

* Visual Studio and Excel
To use the testing helper you should already have Visual Studio 2019 and Excel (any version) installed.
The example will mostly use C#, but Visual Basic is fully supported can also be used for creating your test project.

* xUnit
[xUnit](https://xunit.net/) is a unit testing tool for the .NET Framework. 

If you are not familiar with unit test frameworks, or with xUnit in particular, you might want to look at or work through the XUnit Getting Started instructions for 
[Using .NET Framework with Visual Studio](https://xunit.net/docs/getting-started/netfx/visual-studio).

## Creating a test project
A new test project is started by:
* creating a new  'Class Library (.NET Framework)' project (using C# or Visual Basic) and
* installing the `ExcelDna.Testing` package from the NuGet package manager (currently a pre-release package, so check the relevant checkbox or add the `-Pre` flag to the NuGet command line).

After installing the `ExcelDna.Testing` package, the project will have the xUnit framework and Visual Studio runner for xUnit installed, so no additional packages are needed.

## Testing examples

### *ExcelTest* - A simple Excel test

This is a simple test the exercises Excel to 

```c#
using Xunit;
using ExcelDna.Testing;

[assembly:TestFramework("Xunit.ExcelTestFramework", "ExcelDna.Testing")]

namespace ExcelTest
{
    public class CalculationTests
    {
        [ExcelFact]
        public void NumbersAddCorrectly()
        {
            // Get hold of the Excel instance, create a workbook, and then reference the first sheet
            var app = Util.Application;
            var wb = app.Workbooks.Add();
            var _testSheet = wb.Sheets[1];

            // Write two numbers to the active sheet, and a formula that adds them, together
            _testSheet.Range["A1"].Value = 2.0;
            _testSheet.Range["A2"].Value = 3.0;
            _testSheet.Range["A3"].Formula = "= A1 + A2";

            // Read back the value from the cell with the formula
            var result = _testSheet.Range["A3"].Value;

            // Check that we have the expected result
            Assert.Equal(result, 5.0);
        }
    }
}
```

To run the tests in Visual Studio, open the Test Explorer tool window, check that the test is correctly discovered, and press Run.

#### Discussion

Some notable aspects of the above code snippet:
* The `Xunit.ExcelTestFramework` is configured through the `Xunit.TestFramework` assembly-scope attribute.
* Tests are public instance methods marked by an `[ExcelDna.Testing.ExcelFact]` attribute.
* Test code can access Excel `Application` object with a call to `ExcelDna.Testing.Util.Application`. This will refer to the correct Excel root COM object, whether the test code is running in-process or out-of-process (see below).

### *AddInTest* - Testing an add-in

For this test project we create a simple Excel-DNA add-in with a single UDF, and then implement a test project that exercises the add-in function inside Excel.

> TODO

## Solution layout suggestion

For supporting both functional unit testing and Excel integration testing, one possible solution layout is as follows:

* *Library* - contains the core functionality, e.g. calculations or external data access methods. Does not reference Excel-DNA or Excel.
* *Library.Test* - unit test project for the functionality in `Library`, using the standard `xunit` and `xunit.runner.visualstudio` packages as described in the [xUnit documentation](https://xunit.net/docs/getting-started/netfx/visual-studio).
* *AddIn* - Excel AddIn to integrate the functionality from `Library` into Excel, using the `ExcelDna.AddIn` package. Functions declared here contain Excel-specific attributes and information, and deal with the Excel data types and error values if needed before calling into 'Library' methods.
* *AddIn.Test* - integration testing project for 'AddIn', using the `ExcelDna.Testing` package.

## Reference

### Test helper classes 

These types are declared in the `ExcelDna.Testing` assembly.

* *`Xunit.ExcelTestFramework`* - this is the xUnit integration class, and should be indicated in the assembly-scope TestFramework attribute inside the test project:
```c#
[assembly:TestFramework("Xunit.ExcelTestFramework", "ExcelDna.Testing")]
```

* *`ExcelDna.Testing.ExcelFactAttribute`* - this is the method-scope attribute to indicate that a method implements a test.
```c#
      [ExcelFact]
      public void NumbersAddCorrectly() { ... }
```
* *`ExcelDna.Testing.ExcelTestSettingsAttribute`* - this is a class-scope attribute that configures settings for all tests in a class.
```c#
      [ExcelTestSettings(OutOfProcess=true)]
      public class CalculationTests { ... }
```

* *`ExcelDna.Testing.Util`* - provides access to the root Excel Application object, any pre-loaded Workbook and the directory where the test assembly is located.

### In-process vs out-of-process test running
The test methods can execute in two environments:

* Inside the Excel process (the default) - The test runner will load a helper add-in (called ExcelAgent) into the Excel process, and ExcelAgent in turn will load the test library. Test will then run inside the Excel process, which improves performance and gives access to the the Excel C API - `XlCall.Excel(...)` - from the test code.

* Out-of-Process  - There is also an option to run tests out-of-process. This is indicated by setting the `OutOfProcess` property of the `ExcelFact` or `ExcelSettings` attribute on the methopd or class respectively. In this case the test assembly will run inside the xUnit test runner and communicate with Excel via the stardard cross-process COM interop. In this approach there is no additional test agent loaded into the Excel process.

### Error values - COM vs C API

One of the motivations for doing integration testing of an add-in in Excel is to ensure the behaviour of a function when receiving various unexpected values from Excel is correct. In particular, a function running in Excel might receive input values like 'Empty', 'Missing' or some Excel-specific error value like '#VALUE'. Depending on how the test code is reading values from Excel, these additional data types would be represented in different ways.
