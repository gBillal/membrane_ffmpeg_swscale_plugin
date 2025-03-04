# Membrane FFmpeg SWScale plugin

[![Hex.pm](https://img.shields.io/hexpm/v/membrane_ffmpeg_swscale_plugin.svg)](https://hex.pm/packages/membrane_ffmpeg_swscale_plugin)
[![API Docs](https://img.shields.io/badge/api-docs-yellow.svg?style=flat)](https://hexdocs.pm/membrane_ffmpeg_swscale_plugin/)
[![CircleCI](https://circleci.com/gh/membraneframework/membrane_ffmpeg_swscale_plugin.svg?style=svg)](https://circleci.com/gh/membraneframework/membrane_ffmpeg_swscale_plugin)

Plugin providing an element scaling raw video frames and performing pixel format conversions, using SWScale module of [FFmpeg](https://www.ffmpeg.org/) library.

It is a part of [Membrane Multimedia Framework](https://membrane.stream).

## Installation

Add the following line to your `deps` in `mix.exs`. Run `mix deps.get`.

```elixir
{:membrane_ffmpeg_swscale_plugin, "~> 0.14.0"}
```

The precompiled builds of the [ffmpeg](https://www.ffmpeg.org) will be pulled and linked automatically. However, should there be any problems, consider installing it manually.

### Manual instalation of dependencies

#### macOS

```shell
brew install ffmpeg
```

#### Ubuntu

```shell
sudo apt-get install ffmpeg
```

#### Arch / Manjaro

```shell
pacman -S ffmpeg
```

## Description

### PixelFormatConverter
PixelFormatConverter accepts raw video in any of the pixel formats specified in type [`Membrane.RawVideo.pixel_format_t()`](https://hexdocs.pm/membrane_raw_video_format/Membrane.RawVideo.html#t:pixel_format_t/0)
as input and is capable of producing output in any of these pixel formats.

When creating the element you need to specify a single option `format` defining the desired pixel format of the output.
The element requires `Membrane.RawVideo` stream format on the input with `aligned: true` constraint, meaning that each buffer must contain exactly one raw video frame.

### Scaler
Scaler needs input in the YUV420p format, processes one frame at a time and requires getting stream format with input video
width and height. To meet all requirements either `Membrane.RawVideo.Parser` or some decoder
(e.g. `Membrane.H264.FFmpeg.Decoder`) have to precede Scaler in the pipeline.

There are two options that have to be specified when creating the element:

- `output_width` - desired scaled video width.
- `output_height` - desired scaled video height.

Both need to be even numbers.

Scaling consists of two operations:

- scaling itself - resizing video frame with keeping original ratio. After that operation at least one of the
dimensions of the input frame match the respective dimension of the desired output size. The second one
(if does not match) is smaller than its respective dimension.
- adding paddings - if one dimension does not match after scaling, paddings have to be added. They are put on both
sides of the scaled frame equally. They are either above and below the frame or on the left and right sides of it.
It depends on the dimension that did not match after scaling.

The output of the element is also in the YUV420p format. It has the size as specified in the options.

## Usage

### `Membrane.FFmpeg.SWScale.Scaler`

Scaling an encoded (using H.264 standard) video requires parser and decoder because Scaler scales only raw,
decoded video. The pipeline scales the video and reencodes it.

```elixir
defmodule Scaling.Pipeline do
  use Membrane.Pipeline

  @impl true
  def handle_init(_ctx, _options) do
    structure = [
      child(:file_src, %Membrane.File.Source{location: "/tmp/input.h264"})
      |> child(:parser, Membrane.H264.FFmpeg.Parser)
      |> child(:decoder, Membrane.H264.FFmpeg.Decoder)
      |> child(:scaler, %Membrane.FFmpeg.SWScale.Scaler{output_width: 640, output_height: 640})
      |> child(:encoder, Membrane.H264.FFmpeg.Encoder)
      |> child(:file_sink, %Membrane.File.Sink{location: "/tmp/output.h264"})
    ]

    {[spec: structure}, %{}}
  end
end
```

### `Membrane.FFmpeg.SWScale.PixelFormatConverter`

Converting the pixel format of an encoded (using H.264 standard) video requires a parser and a decoder
because the Converter performs conversion only on the raw , decoded video. The pipeline changes the pixel
format of the video to the `I422` format and reencodes it.

```elixir
defmodule Converting.Pipeline do
  use Membrane.Pipeline

  @impl true
  def handle_init(_ctx, _options) do
    structure = [
      child(:file_src, %Membrane.File.Source{location: "/tmp/input.h264"})
      |> child(:parser, Membrane.H264.FFmpeg.Parser)
      |> child(:decoder, Membrane.H264.FFmpeg.Decoder)
      |> child(:converter, %Membrane.FFmpeg.SWScale.PixelFormatConverter{format: :I422})
      |> child(:encoder, Membrane.H264.FFmpeg.Encoder)
      |> child(:file_sink, %Membrane.File.Sink{location: "/tmp/output.h264"})
    ]
    
    {[spec: structure], %{}}
  end
end
```

## Copyright and License

Copyright 2021, [Software Mansion](https://swmansion.com/?utm_source=git&utm_medium=readme&utm_campaign=membrane)

[![Software Mansion](https://logo.swmansion.com/logo?color=white&variant=desktop&width=200&tag=membrane-github)](https://swmansion.com/?utm_source=git&utm_medium=readme&utm_campaign=membrane)

Licensed under the [Apache License, Version 2.0](LICENSE)
