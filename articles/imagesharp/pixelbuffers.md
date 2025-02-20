# Working with Pixel Buffers

### Setting individual pixels using indexers
A very basic and readable way for manipulating individual pixels is to use the indexer either on `Image<T>` or `ImageFrame<T>`:
```C#
using (Image<Rgba32> image = new Image<Rgba32>(400, 400))
{
    image[200, 200] = Rgba32.White; // also works on ImageFrame<T>
}
```

The indexer is an order of magnitude faster than the `.GetPixel(x, y)` and `.SetPixel(x, y)` methods of `System.Drawing`, but individual `[x, y]` indexing has inherent overhead compared to more sophisticated approaches demonstrated below.

### Efficient pixel manipulation
If you want to achieve killer speed in your pixel manipulation routines, you should utilize the per-row methods. These methods take advantage of the [`Span<T>`-based memory manipulation primitives](https://www.codemag.com/Article/1807051/Introducing-.NET-Core-2.1-Flagship-Types-Span-T-and-Memory-T) from [System.Memory](https://www.nuget.org/packages/System.Memory/), providing a fast, yet safe low-level solution to manipulate pixel data.

This is how you can implement efficient row-by-row pixel manipulation. This API receives a @"SixLabors.ImageSharp.PixelAccessor`1" which ensures that the span is never [transferred to the heap](#spant-limitations) making the operation safe.

> [!Note]
> The pixel manipulation APIs have been changed in ImageSharp 2.0.
> If you are interested about the background of these changes, see the [API discussion on GitHub](https://github.com/SixLabors/ImageSharp/issues/1739).

```C#
using SixLabors.ImageSharp;

// ...
using Image<Rgba32> image = Image.Load<Rgba32>("my_file.png");
image.ProcessPixelRows(accessor =>
{
    // Color is pixel-agnostic, but it's implicitly convertible to the Rgba32 pixel type
    Rgba32 transparent = Color.Transparent;

    for (int y = 0; y < accessor.Height; y++)
    {
        Span<Rgba32> pixelRow = accessor.GetRowSpan(y);

        // pixelRow.Length has the same value as accessor.Width,
        // but using pixelRow.Length allows the JIT to optimize away bounds checks:
        for (int x = 0; x < pixelRow.Length; x++)
        {
            // Get a reference to the pixel at position x
            ref Rgba32 pixel = ref pixelRow[x];
            if (pixel.A == 0)
            {
                // Overwrite the pixel referenced by 'ref Rgba32 pixel':
                pixel = transparent;
            }
        }
    }
});
```

It's possible to simplify the part dealing with `pixelRow` using C# 7.3 `foreach ref`:

```C#
foreach (ref Rgba32 pixel in pixelRow)
{
    if (pixel.A == 0)
    {
        // overwrite the pixel referenced by 'ref Rgba32 pixel':
        pixel = transparent;
    }
}
```

Need to process two images simultaneously? Sure!

```C#
// Extract a sub-region of sourceImage as a new image
private static Image<Rgba32> Extract(Image<Rgba32> sourceImage, Rectangle sourceArea)
{
    Image<Rgba32> targetImage = new(sourceArea.Width, sourceArea.Height);
    int height = sourceArea.Height;
    sourceImage.ProcessPixelRows(targetImage, (sourceAccessor, targetAccessor) =>
    {
        for (int i = 0; i < height; i++)
        {
            Span<Rgba32> sourceRow = sourceAccessor.GetRowSpan(sourceArea.Y + i);
            Span<Rgba32> targetRow = targetAccessor.GetRowSpan(i);

            sourceRow.Slice(sourceArea.X, sourceArea.Width).CopyTo(targetRow);
        }
    });

    return targetImage;
}
```

### Parallel, pixel-format agnostic image manipulation
There is a way to process image data in a pixel-agnostic floating-point format that has the advantage of working on images of any underlying pixel-format, in a completely transparent way: using the @"SixLabors.ImageSharp.Processing.PixelRowDelegateExtensions.ProcessPixelRowsAsVector4(SixLabors.ImageSharp.Processing.IImageProcessingContext,SixLabors.ImageSharp.Processing.PixelRowOperation)" APIs.

This is how you can use this extension to manipulate an image:

```C#
// ...

image.Mutate(c => c.ProcessPixelRowsAsVector4(row =>
{
    for (int x = 0; x < row.Length; x++)
    {
        // We can apply any custom processing logic here
        row[x] = Vector4.SquareRoot(row[x]);
    }
}));
```

This API receives a @"SixLabors.ImageSharp.Processing.PixelRowOperation" instance as input, and uses it to modify the pixel data of the target image. It does so by automatically executing the input operation in parallel, on multiple pixel rows at the same time, to fully leverage the power of modern multi-core CPUs. The `ProcessPixelRowsAsVector4` extension also takes care of converting the pixel data to/from the `Vector4` format, which means the same operation can be used to easily process images of any existing pixel-format, without having to implement the processing logic again for each of them.

This extension offers fast and flexible way to implement custom image processors in ImageSharp. In certain cases (typically desktop apps running on multi-core CPU) the processor-level parallelism might be faster and desirable, but in case of high-load server-side applications it usually hurts throughput. To address this, the level of parallelism can be customized via @"SixLabors.ImageSharp.Configuration"'s @"SixLabors.ImageSharp.Configuration.MaxDegreeOfParallelism" property.

### `Span<T>` limitations
Please be aware that **`Span<T>` has a very specific limitation**: it is a stack-only type! Read the *Is There Anything Span Can’t Do?!* section in [this article](https://www.codemag.com/Article/1807051/Introducing-.NET-Core-2.1-Flagship-Types-Span-T-and-Memory-T) for more details.
A short summary of the limitations:
- Span can only live on the execution stack.
- Span cannot be boxed or put on the heap.
- Span cannot be used as a generic type argument.
- Span cannot be an instance field of a type that itself is not stack-only.
- Span cannot be used within asynchronous methods.

### Exporting raw pixel data from an `Image<T>`
You can use @"SixLabors.ImageSharp.Image`1.CopyPixelDataTo*" to copy the pixel data to a user buffer. Note that the following sample code leads to to significant extra GC allocation in case of large images, which can be avoided by processing the image row-by row instead.
```C#
Rgb32[] pixelArray = new Rgba32[image.Width * image.Height]
image.CopyPixelDataTo(pixelArray);
```

Or:
```C#
byte[] pixelBytes = new byte[image.Width * image.Height * Unsafe.SizeOf<Rgba32>()]
image.CopyPixelDataTo(pixelBytes);
```

### Loading raw pixel data into an `Image<T>`

```C#
int width = ...;
int height = ...;
Rgba32[] rgbaData = GetMyRgbaArray();
using (var image = Image.LoadPixelData(rgbaData, width, height))
{
	// Work with the image
}
```

```C#
int width = ...;
int height = ...;
byte[] rgbaBytes = GetMyRgbaBytes();
using (var image = Image.LoadPixelData<Rgba32>(rgbaBytes, width, height))
{
	// Work with the image
}
```

### OK nice, but how do you get a single pointer or span to the underlying pixel buffer?

That's the neat part, you don't. 🙂 Well, normally.

For custom image processing code written in C#, we highly recommend to use the methods introduced above, since ImageSharp buffers are discontiguous by default. However, certain interop use-cases may require to overcome this limitation, and we support that. Please read the [Memory Management](memorymanagement.md) section for more information.