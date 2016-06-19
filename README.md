# Mork

A ruby [optical mark recognition](http://en.wikipedia.org/wiki/Optical_mark_recognition) (OMR) library aimed at accomplishing two tasks in the context of paper-based, multiple-choice tests and surveys:

1. generating [response sheets](/spec/samples/sheet.jpg) in PDF format
2. detecting the responses provided on the [printed sheet](/spec/samples/sample_gray.jpg) by a human with a pen or a pencil.

## Roadmap

This library is under active development. __Until v.1.0 is reached, the API should be regarded as unstable and bound to change without notice.__

## Assumptions and limitations

Mork is a low-level library, and very much work in progress. It is not, and will likely never be a complete OMR solution. While suggestions and contributions are more than welcome, for the time being several assumptions and restrictions apply.

- the PDF files generated by Mork are intended to be printed on regular printer paper
- the entire response sheet must fit into a single page
- after collecting the responses, a filled-out form should be acquired as a JPEG, PNG, or PDF image by a normal optical scanner or camera (i.e., no specialized equipment is necessary)
- independent of how the sheet is printed and the image is acquired, all internal processing is done on a grayscale version of the bitmap
- the response sheet always contains the following items:
  - registration marks at each page corner
  - a response area containing a list of numbered items (questions)
  - each item contains an arbitrary number of choice “cells”, each marked with a capital letter (A, B, C, ...)
  - the right edge of the response area is reserved for a column of calibration cells that must remain blank
- in addition, the sheet may contain:
  - a bar code along the bottom margin to uniquely identify the sheet
  - a header area to print arbitrary information
- it is entirely up to the user to provide parameters that produce the desired response sheet layout; in particular, making sure that the header elements and choice cells fit in the available space, as Mork does not provide any type of "sanity" check at this time

## Getting started

First, make sure that ImageMagick is installed in your system. Typing `convert in a terminal shell should print out a long help message. If instead you get a “command not found”-like error, you will need to install ImageMagick.

In OS X, `brew` is an excellent package manager:

    brew install imagemagick

In a Debian-based Linux distro (e.g., Ubuntu) you would do:

    sudo apt-get install imagemagick

To create a small ruby project that uses Mork, `cd` into a directory of choice, then execute the following shell commands:

```
mkdir mork_test
cd mork_test
echo "source 'http://rubygems.org'" > Gemfile
echo "gem 'mork'" >> Gemfile
echo "require 'mork'" > mork_test.rb
echo "include Mork" >> mork_test.rb
bundle install
```

Edit the file `mork_test.rb` with the code snippets below, then execute the following shell command to see the result:

    bundle exec ruby mork_test.rb

## Generating response sheets with `SheetPDF`

Response sheets are created through the `Mork::SheetPDF` class. Two pieces of information must be provided to the class constructor to produce a meaningful sheet or, more commonly, a set of similar sheets:

- **content**: what to place specifically on each sheet, such as how many response items and choices, what to write in the header, the number to print as a barcode
- **layout**: the sheet description in terms of element positioning, sizes, margins, spacing, fonts, etc. Layout settings apply equally to all generated sheets

```ruby
s = SheetPDF.new content, layout
```

Let’s look at each argument in turn.

### content

The `content` argument should be a hash like the following:

```ruby
content = {
  # number of items and number of choices per item
  # in this case: 100 items with 5 choices each
  choices: [5] * 100,
  # the response sheet's unique identifier; it is printed
  # in binary form as a barcode at the bottom of the sheet
  barcode: 123456,
  # stuff to print in the header
  header: {
    name: 'John Doe UI354320',
    title: 'A serious, difficult test - 31 December 1999',
    code:  '201.48',
    signature: 'Signature'
  }
}
```

These are the key-value pairs that the `content` hash may contain:

- **choices**: an array of integers, specifiying the number of items in the test (the length of the array) and the number of choices available for each item (the array values); if omitted, the maximum number of items and choices per item allowed by the layout (see below) are printed
- **barcode**: an integer number, the sheet's unique identifier to be printed as a binary barcode along the bottom edge of the sheet; if omitted, no barcode is printed
- **header**: a (sub)hash defining header elements, where each key is the element’s name and each value is the content to be rendered; you can place an arbitrary number of elements in the header, as long as the same element names are also defined in the `layout` (see below); if omitted, no header elements are printed

In most situations, you will want to generate several sheets at once. To do that, simply pass an array of content hashes to the constructor.

```ruby
content = [
  {
    barcode: 1001,
    header: { name: 'John Doe UI354320', code:  '1001'}
  },
  {
    barcode: 1002,
    header: { name: 'Jane Roe UI354321', code:  '1002'}
  }
]
```

The `content` can also be specified by passing a string with the path/name of a YAML file like the following:

```yaml
- barcode: 1001
  header:
    name: 'John Doe UI354320'
    code: '1001'
- barcode: 1002
  header:
    name: 'Jane Roe UI354321'
    code: '1002'
```

### layout

The layout specifies exactly the size and location of choice cells, header elements, and barcode bits; it also specifies parameters to fine tune detection of marked cells and of registration marks (see below).

The layout hash is put together in 3 steps during `SheetPDF` initialization:

1. at first, the layout is generated based on a set of built-in options
2. next, if a file named `layout.yml` exists in the current path, Mork automatically parses it; any values found override the built-in ones
3. finally, values specified in the constructor argument override the existing ones

This is the list of built-in values that the layout is initially based on. Please note that the parameters with an (*) in the comment have no effect on PDF production, but are relevant to OMR scan (see further below).

```yaml
page_size:             # all measurements in mm
  width:        210    # width of the paper sheet
  height:       297    # height of the paper sheet
reg_marks:
  margin:        10    # distance from each page border to registration mark center
  radius:         3    # registration mark radius
  offset:         2    # distance between the search area and each page border (*)
  crop:          20    # size of the registration mark search area (*)
  dilate:         5    # size of a “dilate” filter to get rid of stray noise (0 to skip) (*)
  blur:           2    # size of a gaussian blur filter to smooth overly pixelated registration marks (0 to skip) (*)
  contrast:      20    # minimum contrast between registration mark circles and the surrounding white paper (*)
header:
  title:               # ‘title’ is a label of your choosing; you can add arbitrary header elements
    top:         15    # margin relative to registration frame top side
    left:        15    # margin relative to registration frame left side
    width:      160    # text will be fitted to this width
    height:      12    # text will be fitted to this height
    size:        12    # font size
    box:      false    # if true, header element will be enclosed in a frame
items:
  threshold:      0.75 # mark detection threshold (*)
  columns:        4    # number of columns
  column_width:  44    #
  rows:          30    # number of items per column
  left:          11    # response area margin, relative to reg frame
  top:           55    # response area margin, relative to reg frame
  x_spacing:      7    # horizontal distance between ajacent cell centers
  y_spacing:      7    # vertical distance between ajacent cell centers
  cell_width:     6    # width of each choice and calibration cell
  cell_height:    5    # height of each choice and calibration cell
  max_cells:      5    # the maximum number of choices per question
  font_size:      9    # for the question number and choice letters
  number_width:   8    # width of question number text box
  number_margin:  2    # distance between right side of q num and left side of first choice cell
barcode:
  bits:          38    # the maximum sheet identifier is 2 to the power or bits
  left:          15    # distance between registration frame side and the first barcode bit
  width:          3    # width of each barcode bit
  height:         3    # height of each barcode bit from the registration frame bottom side
  spacing:        4    # horizontal distance between adjacent barcode bit centers
```

A `layout.yml` file may be used on top of the above to always apply settings that you would consider your own defaults. A good example might be to override the built-in A4 paper size with Letter paper width and height. For example, if everthing in the built-in layout fits your needs except for paper size, the `layout.yml` file should contain just the following:

```yaml
page_size:
  width:   215.9
  height:  279.4
```

Finally, the `layout` argument into the `SheetPDF` constructor can be either a hash or a string indicating the path/name of a YAML file.

The specification of header elements is slightly different than all other settings, in that you are free to add any number of elements with arbitrary names, as long as each element is given the following properties: `top`, `left`, `width`, `height`, and `size` (see the built-in title element above). The `box` property is optional. As already noted, header elements can be used in the `content` only if their properties are defined in the `layout`.

In summary, the following sample code shows how to create a 2-page PDF document:

```ruby
content = [
  {
    barcode: 1001,
    header: { name: 'John Doe UI354320', title: 'Final exam', code:  '1001'}
  },
  {
    barcode: 1002,
    header: { name: 'Jane Roe UI354321', title: 'Final exam', code:  '1002'}
  }
]

layout = {
  header: {
    name:  { top:  5, left:  15, width: 160, height:  7, size: 14 },
    title: { top: 15, left:  15, width: 160, height: 12, size: 12 },
    code:  { top: 15, left: 155, width:  35, height: 10, size: 14 }
  }
}

sheet = SheetPDF.new content, layout
s.save 'sheet.pdf'
system 'open sheet.pdf' # this works in OSX
```

## Analyzing response sheets with `SheetOMR`

### Preparing a `SheetOMR` object

Assuming that a person has filled out a response sheet by darkening with a pen the selected choices, and that the sheet has been acquired as an image file, response scoring is performed by the `Mork::SheetOMR` class. Two pieces of information must be provided to the object constructor:

- **path**: mandatory path/filename of the bitmap image (accepts JPG, JPEG, PNG, PDF extensions; a resolution of 150-200 dpi is usually more than sufficient to obtain accurate readings)
- **layout**: same as for the `SheetPDF` class

The following code shows how to create a SheetOMR based on a bitmap file named `image.jpg`:

```ruby
# instantiating the object
s = SheetOMR.new 'image.jpg', 'mylayout.yml'
```

When the object is initialized, Mork attempts to register the image, a necessary step before marking can be performed. To find out if registration succeeded:

```ruby
s.valid?
```

Next, we need to indicate the questions/choices of interest for subsequent operations. For example, the following instructs Mork to evaluate 5 choices for each of the first 50 questions:

```ruby
choices = [5] * 50
s.set_choices choices
```

`choices` is equivalent to the array of integers passed to the `SheetPDF` constructor as part of the `content` parameter (see above).

### Marking the response sheet

We are now ready to mark the sheet:

```ruby
mc = s.marked_choices
```

Since the function `set_choices` returns true only if the sheet is properly registered, you can write something like the following (instead of calling `valid?`):

```ruby
s = SheetOMR.new 'image.jpg'
if s.set_choices [5] * 50
  puts s.marked_choices
else
  puts "The sheet is not registered!"
end
```

If all goes well, the `marked` array will contain 50 sub-arrays, each containing the list of marked choices for that item, where the first cell is indicated by a 0, the second by a 1, etc.

Read the [API documentation](http://www.rubydoc.info/gems/mork) to learn about additional methods to extract marked responses.

### Applying overlays on the original image

It is also possible to show the scoring graphically by applying an overlay on top of the scanned image.

```ruby
s.overlay :check, :marked
s.save 'marked_choices.jpg'
system 'open marked_choices.jpg' # this works in OSX
```

More than one overlay can be applied on a sheet. For example, in addition to checking the marked choice cells, the expected choices can be outlined to show correct vs. wrong responses:

```ruby
correct = [[3], [0], [2], [1], [2]] # and so on...
s.overlay :outline, correct
s.overlay :check, :marked
s.save 'marked_choices_and_outlines.jpg'
system 'open marked_choices_and_outlines.jpg' # this works in OSX
```

Scoring can only be performed if the sheet gets properly registered, which in turn depends on the quality of the scanned image.

### Improving sheet registration and marking

For Mork to be able to register the response sheet, the acquired image must be of _sufficient_ resolution and contrast, and it should be _reasonably_ straight, with some white margin around the 4 registration circles.

Mork tries to be tolerant of variations in the above parameters, but you should experiment with your own layout, actual printout, and actual scans.

When registration fails, it is possible to get some information by displaying the status and by applying a dedicated overlay on the original image:

```ruby
s = SheetOMR.new 'image.jpg', 'mylayout.yml'
unless s.valid?
  s.save_registration 'unregistered.jpg'
end
```

Experiment with the following layout settings to improve sheet registration:

```yaml
reg_marks:         
  radius:    3 # a bigger registration mark can sometimes help
  crop:     20 # make sure the registration mark lies comfortably
               # inside the crop area
  offset:    2 # increase the offset if a black background is
               # visible along the page edges
  blur:      2 # increase gaussian blur filter size if images are
               # noisy
  dilate:    5 # higher values can remove more specks and dust from
               # the crop areas, but may also “eat out” too much of
               # the registration mark
  contrast: 20 # decrease this value if the contrast between the
               # registration marks’ black and the surrounding white
               # paper is low
```

In addition, you can fine tune the threshold for accepting a choice as marked:

```yaml
items:
  threshold: 0.75 # higher values will result in fewer false positives, but
                  # lightly darkened choices may be missed; conversely, lower
                  # values will reduce misses but may increase false positives
```

If Mork fails to register/mark scoring sheets that you believe **should** be valid, please open an issue and attach a sample image.

## API Reference
The online rubygems docs are [here](http://www.rubydoc.info/gems/mork)

_(Wait, why Mork? Because I used to have a crush on Mindy)_
