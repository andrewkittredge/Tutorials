# Excel-DNA and Dynamic Arrays

The support for 'Dynamic Arrays' is a major new Excel feature available in the Office 365 versions of Excel since 2020.
There are many excellent resources available that explore the power of dynamic arrays - I provide a few links below as introduction to the topic.
In this tutorial I will show how dynamic arrays interact with user-defined functions defined in Excel-DNA add-ins.

## Background

For this tutorial I will assume you are already familiar with the fundamentals of Excel-DNA and the dynamic arrays feature in Excel.
For general background on dynamic arrays, there are many excellent introductions available - I suggest a few below.

#### Dynamic Array links

[Excel dynamic arrays, functions and formulas by Svetlana Cheusheva from AbleBits](https://www.ablebits.com/office-addins-blog/2020/07/08/excel-dynamic-arrays-functions-formulas/) provides a great introduction to dynamic arrays.

Some of Microsoft's notes on Dynamic Arrays:
* [Preview of Dynamic Arrays in Excel](https://techcommunity.microsoft.com/t5/excel-blog/preview-of-dynamic-arrays-in-excel)
* [Dynamic arrays and spilled array behavior](https://support.office.com/en-us/article/dynamic-arrays-and-spilled-array-behavior-205c6b06-03ba-4151-89a1-87a7eb36e531)
* [Dynamic array formulas in non-dynamic aware Excel](https://support.office.com/en-us/article/dynamic-array-formulas-in-non-dynamic-aware-excel-696e164e-306b-4282-ae9d-aa88f5502fa2)
* [Implicit intersection operator: @](https://support.office.com/en-us/article/implicit-intersection-operator-ce3be07b-0101-4450-a24e-c1c999be2b34)

Bill Jelen (Mr. Excel) goes into the topic in great details in an [e-book about Dynamic Arrays](https://www.mrexcel.com/products/excel-dynamic-arrays-straight-to-the-point-2nd-edition/), and 
 a [Youtube video](https://youtu.be/ViSEZLPmRvw) showing how powerful the dynamic arrays and mathcing new formulas are.

[Formula vs Formula2](https://docs.microsoft.com/en-us/office/vba/excel/concepts/cells-and-ranges/range-formula-vs-formula2)

And countless more write-ups and videos on YouTube.

## Dynamic arrays and Excel-DNA user-defined functions

The overall message is that dynamic arrays Excel and Excel-DNA add-ins work very well together.
Making add-in functions 'array-friendly' provide elegant solutions to problems that required awkward workarounds in older Excel versions.

### Returning arrays

I'll start with a simple function that returns an array result - just some strings with an array size that is determined by the input parameters.

```cs
public static object dnaMakeArray(int numRows, int numCols)
{
    object[,] result = new object [rows, columns];
    for (int i = 0; i < rows; i++)
    {
        for (int j = 0; j < columns; j++)
        {
            result[i,j] = $"{i}|{j}";
        }
    }
    return result;
}
```

Used in a dynamic arrays version of Excel, the result will automatically spill to the right size.
As the inputs change (the number of rows and columns) the resulting spill region automatically resizes.

A few things to note:
* Arrays with 0 size produce an error
* Arrays with a single element don't result in a spill region (no shadow / blue border in the cell)
* If there is no room to spill, we get a `#SPILL!` error

### Array inputs

Next we look at a simple function that describes its single input value.

```cs

```

We can see the following:
* ??? simple values

The `AddThem` starter function taking two numbers and adding, would look like this.
```cs
public static double dnaAddThem(double val1, double val2)
{
    return val1 + val2;
}
```

Let's build an array-aware version of the `AddThem` starter function.
Excel-DNA helps simplify the function a bit when we make the input parameters of type double[,] or object[,] - even with single values we'll get a 1x1 array, so the processing can be more uniform.

```cs
public static double[,] dnaAddThemArrays(double[,] val1, double[,] val2)
{
    // if the inputs are not the same size, we return throw na exception, which returns #VALUE back to Excel
    int rows1 = val1.GetLength(0);
    int cols1 = val1.GetLength(1);
    int rows2 = val2.GetLength(0);
    int cols2 = val2.GetLength(1);
    
    if (rows1 <> rows2 || cols1 <> cols2)
        throw new ArgumentException("Incompatible array sizes");
    
    double[,] result = new double[rows1, cols1];
    for (int i = 0; i < rows; i++)
    {
        for (int j = 0; j < columns; j++)
        {
            result[i,j] = val1[i,j] + val2[i,j];
        }
    }
    return result;
}
```

**NOTE:** One danger of using `double` input parameters is that Excel will convert empty cells to 0-values.
To be more careful about the exact input types, change the parameter types to `object[,]` and check the input types during processing.

It's possible to make a general-purpose function that transforms a single-input, single-output function like `dnaAddThem` into an array-aware version.
With such a helper, we can write a formula like `=ARRAY.MAP2(dnaAddThem, A1:A10, B1:B10)` where we pass the single-valued `dnaAddThem` function without parentheses into the transformation function, where it will be called for every pair of inputs.
The [`ArrayMap`](https://github.com/Excel-DNA/Samples/tree/master/ArrayMap) sample project explores in more details how this can be done.

### Implicit intersection

Let's now look at implicit intersection and the @-operator, and how these work with Excel-DNA functions.

???

### `ExcelReference` inputs and results

There are some cases where we don't need to know the input values, but can provide processing based on the input array size.
An example would be a function that returns only the first few rows of an array:

```cs
public static object dnaGetArrayHead([ExcelArgument(AllowReference=true)] object input, int numRows)
{
    if (input is ExcelReference inputRef)
    {
        var rowFirst = inputRef.RowFirst;
        var rowLast = Math.Min(inputRef.RowFirst + numRows, inputRef.RowLast);
        return new ExcelReference(rowFirst, rowLast, inputRef.ColumnFirst, inputRef.ColumnLast, inputRef.SheetId);
    }
    else if (input is object[,] inputArray)
    {
        var rows = inputArray.GetLength(0);
        var cols = inputArray.GetLength(1);
        
        var resultRows = Math.Min(rows, numRows);
        var result = new object[resultRows, cols];
        for (int i = 0; i < resultRows; i++)
        {
            for (int j = 0; j < cols; j++)
            {
                result[i, j] = inputArray[i, j];
            }
        }
        return result;
    }
    else
    {
        // Just a scalar value
        if (numRows >= 1)
        {
            return input;
        }
    }
    // Otherwise we have an error - return #VALUE!
    return ExcelError.ExcelErrroValue;
}
```

### RTD-based async and streaming functions

Under dynamic arrays Excel, async and streaming functions that are implemented in Excel-DNA with the RTD feature, can return array results without any problems.
In previous versions of Excel there were sometimes problems when arrays are returned in these cases, and the results could not be resized automatically.
These issues are now successfully resolved.

A small example of an async function returning an array after a specific delay:

```cs
public static object dnaMakeArrayAsync(int delayMs, int numRows, int numCols)
{
    var funcName = nameof(dnaMakeArrayAsync);
    var args = new object[] {delayMs, numRows, numCols};
    
    return ExcelAsyncUtil.Run(funcName, args, () =>
    {
        Thread.Sleep(delayMs);
        object[,] result = new object [rows, columns];
        for (int i = 0; i < rows; i++)
        {
            for (int j = 0; j < columns; j++)
            {
                result[i,j] = $"{i}|{j}";
            }
        }
        return result;
    });
}

```

## Compatibility with non-dynamic arrays Excel versions

To ensure compatibility in the calculation results for a workbook between DA Excel and pre-DA Excel, some formulas are transformed (actually displayed in different ways) between the versions. This particularly affects the new array-related built-in functions added in DA Excel, and any UDFs used (VBA or Excel-DNA based UDFs).

Looking at the VersionCompare.xlsx workbook we see:
* ??? @ operator
* ??? _xlfn
* ??? non-resizing
* ??? Ctrl_Shift+Enter -> new

Further below I discuss a partial implementation of resizing arrays that can be used in older Excel.

### COM Object Model - `Range.Formula2` to avoid '@'-formulas; `HasSpill` and `SpillRange`

The COM object model has been extended with some extra members added to the `Range` object related to dynamic arrays.
The first things to note is that a formula containing any UDF added to Excel like this:
```vb
Application.Range("A1").Formula = "=dnaAddThem(1,2)"
```
will be processed and displayed as `=@dnaAddThem(1,2)`, with the @-implicit intersection operator added.
This transformation of the formula ensures that the code setting `Range.Formula` has exactly the same effect in older Excel versions as in DA Excel.

DA Excel adds a new `Formula2` member to the `Range` object to set formulas that are compatible with dynamic arrays, e.g.
```vb
Application.Range("A1").Formula = "=dnaAddThem(1,2)"
```

Further information is available on the Microsoft page discussing [`Formula vs. Formula2`]([Formula vs Formula2](https://docs.microsoft.com/en-us/office/vba/excel/concepts/cells-and-ranges/range-formula-vs-formula2)
).

Further Range information properties have been added to get information about the Spill range of a cell.
```vb
Application.Range("A1").HasSpill
Application.Range("A1").SpillRange
```

### 'Classic' ArrayResizer

Before the advent of dynamic arrays in Excel, I created a helper function to manage array results from functions called the `ArrayResizer`.
Find more details in the [ArrayResizer sample on GitHub](https://github.com/Excel-DNA/Samples?arrayResizer).

#### Testing for whether the running Excel instance supports dynamic arrays

In the ArrayResizer sample, and otherwise, it might be useful to know if an Excel instance supports dynamic arrays.
This (hidden) function check to the presence of the `=FILTER` built-in function to determine this:

```cs
        static bool? _supportsDynamicArrays;  
        [ExcelFunction(IsHidden=true)]
        public static bool dnaSupportsDynamicArrays()
        {
            if (!_supportsDynamicArrays.HasValue)
            {
                try
                {
                    var result = XlCall.Excel(614, new object[] { 1 }, new object[] { true });
                    _supportsDynamicArrays = true;
                }
                catch
                {
                    _supportsDynamicArrays = false;
                }
            }
            return _supportsDynamicArrays.Value;
        }
```