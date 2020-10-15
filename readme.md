# Fast Html Content Builder


## Introduction

When you need to construct HTML dynamically, you can use an [IHtmlContentBuilder](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.ihtmlcontentbuilder). This newly introduced concept helps you build HTML safely, as it has two main entry points: `Append` and `AppendHtml`, that distinguish between already encoded HTML and unencoded text segments (or "entries"). While the default implementation, [HtmlContentBuilder](https://github.com/dotnet/aspnetcore/blob/master/src/Html/Abstractions/src/HtmlContentBuilder.cs) works well for ordinary use casees, it may not suit same advanced ones. Let's see why.

An example of usage looks like this:
```cs
public void RenderBold(this IHtmlContentBuilder builder, string text)
{
	builder.AppendHtml("<b>");
	builder.Append(text); // should be encoded
	builder.AppendHtml("</b>");
}
```

You must be familiar with another concept called [IHtmlContent](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.ihtmlcontent), which represents raw HTML content (that doesn't have to be encoded) in the domain of ASP.NET Core. And since it is optimized for write, you can't read its content (other than let it write to a buffer), but only write it to somewhere using [WriteTo](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.ihtmlcontent.writeto).
If you look at the implementation of [HtmlContentBuilder](https://github.com/dotnet/aspnetcore/blob/master/src/Html/Abstractions/src/HtmlContentBuilder.cs), it doesn't write its content immediately, but it only stores the appended segments, which can be any of the following kind:
 - `String`, or
 - [IHtmlContent](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.ihtmlcontent)

Since they have no common ancestor, the implementation uses a list of `object`s as a storage:
```cs
internal IList<object> Entries { get; }
```

And this storage is used only when the final content is rendered (simplified implementation):
```cs
public void WriteTo(TextWriter writer, HtmlEncoder encoder)
{
	foreach (var entry in Entries)
	{
		switch (entry)
		{
			case string entryAsString:
				// encode and write
				encoder.Encode(writer, entryAsString);
				break;

			case IHtmlContent entryAsHtmlContent:
				// write child
				entryAsHtmlContent.WriteTo(writer, encoder);
				break;
		}
	}
}
```

### The use case
I wanted to create a Markdown to HTML compiler as efficient as possible, and this use case consists of the following usage patterns:
 - append constant HTML blocks, e.g.: `<b>`
 - append parts of an already existing text, e.g.: from the text `this is **bold**` the part `[10..14]` (`bold`)
 - cache compiled final result in memory

### Allocation when writing HTML
Your intuition may say that it is cheaper to write raw HTML, because it doesn't have to be processed further, than text that has to be encoded. This is true in terms of processing, but it is not in terms of memory.

The (simplified) implementation looks like this:
```cs
public void AppendHtml(string encoded)
{
    Entries.Add(new HtmlString(encoded)); // allocation to heap
}
```

Every time you append raw HTML, a new instance of [HtmlString](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.htmlstring) wraps it around - that is allocated to the heap. So, even if you append constant strings as HTML, each write creates future work for the Garbage Collector.

### Allocation when copying strings
If the text you want to write doesn't exist on its own, but it is only a part of a bigger text, you may use [StringSegment](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.stringsegment) to spare copying that substring into a new string. Some example use cases when partial results are not important to exist as invidual, whole strings:
 - a text highlighter, that highlights search matches and compiles into highlighter and not highlighted segments
 - a Markdown compiler that parses Markdown and compiles into HTML

But unfortunately, the default implementation doesn't offer an overload with a `StringSegment`, or its more general form `ReadOnlySpan<char>`:
```cs
IHtmlContentBuilder Append(ReadOnlySpan<char> span);
```

It accepts only `String`:
```cs
IHtmlContentBuilder Append(string unencoded);
```

So if you would call the basic `string` overload with a `StringSegment`, you would end up with an implicit conversion to `string`, and by that, lose the goal to save allocations on heap:
```cs
public void RenderBold(this IHtmlContentBuilder builder, StringSegment text)
{
	builder.AppendHtml("<b>");
	builder.Append(text); // implicit conversion to String!
	builder.AppendHtml("</b>");
}
```

And even if it would offer an overload with the current storage mechanism, it would still not work but at another level: by boxing a value type as a reference type to be able to add it to a list of objects:
```cs
public IHtmlContentBuilder Append(ReadOnlySpan<char> span)
{
	Entries.Add(span); // box to Object
	return this;
}
```

### Allocation when rendering
When you have finished writing, to get the content as a whole is a little complicated, because you can only write it to a `TextWriter`:
```
public string ToString(this IHtmlContentBuilder builder)
{
	using var writer = new StringWriter();
	builder.WriteTo(writer, HtmlEncoder.Default);
	return writer.ToString();
}
```

Note that, you can't even estimate the required output buffer size to render the content, because half of the content of a builder are `String`s (which has `Length`), but the other half are `IHtmlContent`s, that has no capability to get its length, its only capability is [WriteTo](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.html.ihtmlcontent.writeto), write it to a `TextWriter` - where we face the same problem: can't estimate how long it is going to be. So, it is very likely that when we try to connect the pieces together, we end up with resizing the initial buffer size multiple times as the content outgrows is. And these allocations would create a lot of garbage.


## Solution
To make an allocation-free builder, the solution is more complex than simply changing a small part in the implementation. It needs a shift in the concept as well, but it is going to be more well crafted for some other use cases. We are going to use storage pooling at multiple layers.

### Buffer
At first, we are going to buffer writes - so the final result - using an [ArrayBufferWriter](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraybufferwriter-1).
```cs
public class ArrayBufferHtmlContentBuilder : IHtmlContentBuilder
{
	public ArrayBufferHtmlContentBuilder()
	{
		Writer = new ArrayBufferWriter<char>();
	}

	private ArrayBufferWriter<char> Writer { get; }
}
```

The concept of [ArrayBufferWriter](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraybufferwriter-1) is actually pretty simple: it serves as a guide for filling an array incrementally with data for some extent.

### Writing raw HTML
In this case, writing raw HTML is simple, because we only need to copy the input into the buffer:
```cs
public void AppendHtml(ReadOnlySpan<char> span)
{
	Writer.Write(span); // extension method
}
```

We don't even need other overloads, because:
 - `ReadOnlyChar<span>` is the most general way to express a span of characters stored somewhere, and it is a value type
 - `StringSegment` can be implicitly converted to a `ReadOnlyChar<span>`, without copying
 - `String` can be used as a backing storage for a `ReadOnlyChar<span>`, so even if `String` is a reference type, `ReadOnlyChar<span>` is not, so no heap allocation happens

What if we run out of space? In this implementation: we get an exception during copy. This is because the extension method `Write` calls `GetSpan()` without an argument. And in that case, it returns only the remaining space in the underlying array buffer. We can replace the extension method call into a slightly more complicated implementation, that passes a `sizeHint` to the `GetSpan` call. In this case, the `ArrayBufferWriter` is going to check whether there is enough space in the buffer, and if it is not, it is going to resize the underlying array to a bigger one to fit the space needed for the write request.
```cs
public void AppendHtml(ReadOnlySpan<char> span)
{
	var target = Writer.GetSpan(sizeHint: span.Length);
	span.CopyTo(target);
	Writer.Advance(span.Length);
}
```

### Writing unencoded text
This part is a little tricky. Because if the encoder would request a `string` as an input and would return the encoded result as another `string`, than our goal would be hopeless, as we would allocate 2 objects on the heap, just for encoding a small part of a text:
```cs
public void Append(this IHtmlContentBuilder builder, StringSegment text)
{
	builder.AppendHtml(HtmlEncoder.Encode(text)); // 2 new String allocations
}
```

Of course, at first, we need an encoder, that is built in, called [HtmlEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.htmlencoder):
```cs
public HtmlEncoder Encoder { get; }
```

Fortunately, [HtmlEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.htmlencoder) provides a highly efficient implementation, [Encode](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder.encode) for encoding, which accepts a `ReadOnlySpan<char>` as source and a `Span<char>` as a destination. This is exactly what we need.
```cs
public void Append(ReadOnlySpan<char> unencoded)
{
    var span = Writer.GetSpan();
    Encoder.Encode(unencoded, span, out _, out var written);
    Writer.Advance(written);
}
```

In this implementation, there are no heap allocations. But what if we run out of space? As we've learnt before, this implementation would not automatically adjust the size of the buffer. But since we don't know how long it is going to be the encoded output, what size of destination `Span<char>` do we need? Less people know, that the [TextEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder) has some hidden features, like a property called [MaxOutputCharactersPerInputCharacter](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder.maxoutputcharactersperinputcharacter), which won't show up in IntelliSense code completion, but still exists. These features are hidden using an attribute, that Visual Studio recognizes:
```cs
[EditorBrowsable(EditorBrowsableState.Never)]
public abstract int MaxOutputCharactersPerInputCharacter { get; }
```

So, to provide a safe implementation, that works even in the worst case, we could request a buffer large enough to write:
```cs
public void Append(ReadOnlySpan<char> unencoded)
{
	var span = Writer.GetSpan(sizeHint: unencoded.Length * Encoder.MaxOutputCharactersPerInputCharacter);
	Encoder.Encode(unencoded, span, out _, out var written);
	Writer.Advance(written);
}
```

This works in all cases, but can be littering. This value in `HtmlEncoder` is `10`, so every time we need to encode a small part of a text, we would request a buffer ten times the size of the original input. And this is the worst case, when all characters need to be encoded, and the encoded form is the longest possible.

One of the things we could try is a little less scientific, but can work in practice: in most real world use cases, we don't have to encode all characters, only a few, and the encoded characters won't be 10 characters long. So, we could choose a lower factor, like `2` (output is two times as long as input), try encode, let's see whether it fits (it will, most of the time), and if it doesn't, retry with the maximum sized buffer. It would be fast and efficient most of the time, and rarely it is going to be worse, while the worst case is worse anyway.

Another optimalization we can do is check whether we need to encode at all. In many cases, especially if the text is in English, and doesn't contain special characters, its encoded form is exactly the same as its unencoded. There is another hidden method [FindFirstCharacterToEncode](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder.findfirstcharactertoencode) to help with that:

And we could use that like this:
```cs
public void Append(ReadOnlySpan<char> unencoded)
{
	unsafe
	{
		fixed (char* ptr = unencoded)
		{
			if (Encoder.FindFirstCharacterToEncode(ptr, unencoded.Length) == -1)
			{
				AppendHtml(unencoded);
				return;
			}
		}
	}

	// encode
}
```

Why does it worth checking whether we need to encode at all?
 - of course, we can shortcut those cases, when the input is trivial
 - we don't have to oversize the output buffer to fit the worst case, which end usually remains empty
 - We could just let all characters flow through `Encode` and let it copy them as they are to the output, right? Yeah, but `Encode` is slow, because it is sequential, while the other path can use Hardware Intrinsics completely covered, which can make it really really fast:
   - [`TextEncoder.FindFirstCharacterToEncodeUtf8`](https://github.com/dotnet/runtime/blob/master/src/libraries/System.Text.Encodings.Web/src/System/Text/Encodings/Web/TextEncoder.cs) supports vectorization to check whether we need to encode
   - `ReadOnlySpan.CopyTo` supports vectorization for copying (which is our fallback to `AppendHtml`)

### Render output
Once we have finished writing, we want to read back the whole content. The `ArrayBufferWriter` keeps track of what was written, there is a property called [WrittenSpan](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraybufferwriter-1.writtenspan) which marks the segment of the underlying array that was used so far. A `String` can be easily constructed from that:
```cs
public override string ToString() => new String(Writer.WrittenSpan);
```

And actually this is the only place where we can't spare the allocation to the heap, when we render the output into a `String`.

If we want to cache this result for future use, we should do that directly as `IHtmlContent`, to avoid further allocations, so let's add a method for creating one:
```cs
public HtmlString ToHtmlString() => new HtmlString(ToString());
```

And of course, the original intent of `IHtmlContentBuilder` can be implemented as well, to write the result to a `TextWriter`, fortunately it accepts `ReadOnlySpan<char>` directly:
```cs
public void WriteTo(TextWriter writer, HtmlEncoder encoder) => writer.Write(Writer.WrittenSpan);
```

Note: this is where the original concept bends a little and it shows up on the interface. While the original idea is to collect pieces and render only at the end, in that case the `HtmlEncoder` can be provided late, at the time of render. While in our concept, we render the output right at the time of each write, so we need an `HtmlEncoder` early, and it can't be replaced later. But I think this trade-off is feasible to support the original interface by ignoring the passed encoder, as 1) the way of encoding HTML is not something that change during runtime, and 2) for testing purposes a fake encoder can be provided in the constructor.

### Pooling the builders
After using a builder, and its underlying buffer has grown to a decent size (but still having lots of empty space as well), we should not throw out the builder, as we would lose a lot of memory that has been already allocated during the writes. Instead, we should make the builder reusable.

The `ArrayBufferWriter` which is the soul of our implementation can be easily cleared:
```cs
public void Clear()
{
	Writer.Clear();
}
```

After calling this, our builder can be used again. And as far the buffer lasts, we would not need to resize it either, so we can be completely allocation free.

And by implementing an [IObjectPoolPolicy<ArrayBufferHtmlContentBuilder>](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.ipooledobjectpolicy-1), we could easily create an [ObjectPool<ArrayBufferHtmlContentBuilder>](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.defaultobjectpool-1) to store reusable instances.
```cs
public sealed class ArrayBufferHtmlContentBuilderPoolPolicy : IPooledObjectPolicy<ArrayBufferHtmlContentBuilder>
{
	public ArrayBufferHtmlContentBuilder Create() => new ArrayBufferHtmlContentBuilder(HtmlEncoder.Default);

	public bool Return(ArrayBufferHtmlContentBuilder obj)
	{
		obj.Clear();
		return true;
	}
}
```

And with this, we have a builder, that is allocation free after the first few usages, when it has to adjust the buffer size to some longer payloads.


## Appendix

### Write URL parts
When we dynamically generate HTML, sometimes we need to dynamically generate parts of URLs as well, and to do that, we should not make the same mistake as before:
```cs
public void RenderYouTube(this IHtmlContentBuilder builder, StringSegment videoId)
{
	builder.AppendHtml("<iframe src=\"https://www.youtube.com/embed/");
	builder.Append(UrlEncoder.Encode(videoId)); // 2 new String allocations
	builder.AppendHtml("\"></iframe>");
}
```

In the example above, we would:
 - convert the `StringSegment videoId` to a `String` to be able to pass it `Encode`
 - `Encode` would create the output as a new `String`

And this is tricky again, because we need to encode two times:
 - first, we need to make sure that data we want to put into the URL remains data and doesn't mess up the URL itself (if it would contain a `?` or `/`), so we need to URL encode it
 - second, we need to make sure that the encoded URL remains data in HTML and doesn't mess it up either, so we need to HTML encode it then

Fortunately, [UrlEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.urlencoder) is a `TextEncoder`, so it provides the same high performance options like `HtmlEncoder`. But to make it easy to use, we can't give the complexity of buffer management to the user that simply wants to write a part of a string properly encoded.

We can't write the output of URL encoding continuing the HTML buffer, because then the input and output of the HTML encoding would overlap each other, which is not allowed. So, we need to find another option:
 - if the text to be encoded is usually small, we could allocate a new temporary buffer on the stack using `stackalloc` which gets destroyed immediately after use
 - if the text is long, we could use an `ArrayPool<char>` to rent longer, reusable buffers
 - dedicate an instance level buffer `char[]` for URL encoding (resize if needed), as the builder is not thread-safe anyway, and if we pool the instances, the buffer can be kept alive across writes
 - request a sub-buffer large enough from the HTML buffer, to fit both the URL encoded output and then its HTML encoded output. It is not as crazy as it may sound, because it is very likely that the URL data written to the final HTML buffer is going to be followed by HTML segments that are close to the length (or longer) of the URL encoded partial result.

### Encoding
Today, finally UTF-8 is the common standard encoding that is used everywhere. It has a distinguished place in .NET to speed up the majority of use cases. TextEncoder has specialized methods for [EncodeUtf8](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder.encodeutf8) and [FindFirstCharacterToEncodeUtf8](https://docs.microsoft.com/en-us/dotnet/api/system.text.encodings.web.textencoder.findfirstcharactertoencodeutf8), so we could further optimize our implementation by make use of these.

But to be able to use these kind of optimizations natively, we would need to be able to view a `String` as a `ReadOnlySpan<byte>` encoded in UTF-8, which is not possible without copying at the moment. This request is tracked here: https://github.com/dotnet/csharplang/issues/2212

## References
Full source code for HtmlContentBuilder:
https://github.com/dotnet/aspnetcore/blob/master/src/Html/Abstractions/src/HtmlContentBuilder.cs

Full source code for TextEncoder:
https://github.com/dotnet/runtime/blob/master/src/libraries/System.Text.Encodings.Web/src/System/Text/Encodings/Web/TextEncoder.cs
