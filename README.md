# Mork

A ruby [optical mark recognition](http://en.wikipedia.org/wiki/Optical_mark_recognition) (OMR) library aimed at accomplishing two tasks in the context of paper-based, multiple-choice tests and surveys:

1. generating [response sheets](/spec/samples/sheet.jpg) in PDF format
2. capturing the responses provided on the [printed sheet](/spec/samples/sample_gray.jpg) by a human with a pen or a pencil.

## Assumptions and limitations

Mork is a low-level library, and very much work in progress. It is not, and will likely never be a complete OMR solution. While suggestions and contributions are more than welcome, for the time being several assumptions and restrictions to what the library is capable of apply.

- the PDF files generated by Mork are intended to be printed on regular printer paper
- the entire response sheet must fit into a single page
- after collecting the responses, a filled-out form should be acquired as a JPEG, PNG, or PDF image by a normal optical scanner or camera (i.e., no specialized equipment is necessary)
- independent of how the sheet is printed and the image is acquired, all internal processing is done on a grayscale version of the bitmap
- the response sheet contains the following items:
  - registration marks at each page corner
  - a bar code along the bottom margin to uniquely identify the sheet
  - a header area to print arbitrary information
  - a response area containing a list of numbered items (questions)
  - each item contains an arbitrary number of choice “cells”, each marked with a capital letter (A, B, C, ...)
  - the right edge of the response area is reserved for a column of calibration cells that must remain blank
- it is entirely up to the user to provide parameters that produce the desired response sheet layout; in particular, making sure that the header elements and choice cells fit in the available space, as Mork does not provide any type of "sanity" check at this time

## Getting started

First, make sure that ImageMagick is installed in your system. In OS X, `brew` is the preferred package manager:

    brew install imagemagick

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

Response sheets are created through the `Mork::SheetPDF` class. Two pieces of information must be provided to the class constructor to produce a meaningful object:

- **content**: what to place on the sheet, such as how many response items and choices, what to write in the header, the number to print as a barcode
- **layout**: sizes, margins, spacing, fonts, etc., of each of the printed elements

Let’s look at each argument in turn.

### content

The `content` argument should be a hash like the following:

```ruby
content = {
  # the response sheet's unique identifier; it is printed
  # in binary form as a barcode at the bottom of the sheet
  barcode: 123456,
  # number of items and number of choices per item
  # in this case: 100 items with 5 choices each
  choices: [5] * 100,
  # stuff to print in the header
  header: {
    name: 'John Doe UI354320',
    title: 'Anatomy and Physiology test, Sept 20, 2014',
    code:  '201.48',
    signature: 'Signature'
  }
}
```

These are the key-value pairs that the `content` hash should contain:

- **barcode**: an integer number, the sheet's unique identifier
- **choices**: an array of integers, specifiying the number of items in the test (the length of the array) and the number of choices available for each item (the array values)
- **header**: a (sub)hash of key-value pairs defining the content of named header elements; in each pair, the key is the name of one header element, while the value is the rendered content; the actually available elements are defined in the `layout` (see below)

### layout

The layout argument is also defined as a hash, but because of its length and since the layout often stays identical across many response sheets, it is usually more convenient to write the information in a YAML file and pass its path/filename to the `SheetPDF` constructor instead. Here is the YAML version of a standard layout hash. Please note that the parameters with an (*) in the comment have no effect on PDF production, but are relevant to OMR scan (see further below).

```yaml
page_size:                # all measurements in mm
  width:          210     # width of the paper sheet
  height:         297     # height of the paper sheet
reg_marks:         
  margin:          10     # distance from each page border to registration mark center
  radius:           3     # registration mark radius
  crop:            20     # size of the registration mark search area (*)
  offset:           2     # distance between the search area and each page border (*)
  blur:             2     # size of a gaussian blur filter to smooth overly pixelated registration marks (0 to skip) (*)
  dilate:           5     # size of a “dilate” filter to get rid of stray noise (0 to skip) (*)
  contrast:        20     # minimum contrast between registration mark circles and the surrounding white paper
header:           
  name:                   # ‘name’ is just a label; you can add arbitrary header elements
    top:            5     # margin relative to registration frame top side
    left:          15     # margin relative to registration frame left side
    width:        160     # text will be fitted to this width
    height:         7     # ...and this height
    size:          14     # font size
  title:
    top:           15
    left:          15
    width:        160
    height:        12
    size:          12
  code:
    top:           35
    left:         130
    width:         57
    height:        10
    size:          14
  signature:
    top:            30
    left:           15
    width:         120
    height:         15
    size:            7
    box:          true    # header element will be enclosed in a box
items:
  threshold:         0.75 # mark detection threshold
  top:              55    # response area margin, relative to reg frame
  left:             11    # response area margin, relative to reg frame
  rows:             30    # number of items per column
  columns:           4    # number of columns
  column_width:     44    # 
  x_spacing:         7    # horizontal distance between ajacent cell centers
  y_spacing:         7    # vertical distance between ajacent cell centers
  cell_width:        6    # width of each choice and calibration cell
  cell_height:       5    # height of each choice and calibration cell
  max_cells:         5    # maximum number of choices per item
  font_size:         9    # size of both the item number and choice cell letter
  number_width:      8    # 
  number_margin:     2    # margin between
barcode:
  bits:             40    # the maximum sheet identifier is 2 to the power or bits
  left:             15    # distance between registration frame side and the first barcode bit
  width:             3    # width of each barcode bit
  height:            3    # height of each barcode bit from the registration frame bottom side
  spacing:           4    # horizontal distance between adjacent barcode bit centers
```

Assuming that the above text is written in a file named `layout.yml`, here's how to create a `SheetPDF` object and write the PDF file to disk:

```ruby
s = SheetPDF.new content, 'layout.yml'
s.save 'sheet.pdf'
system 'open sheet.pdf' # this works in OSX
```

It's easy to see that by iterating over a series of `content` hashes you can produce any number of sheets, all based on the same layout but each containing unique information (notably the barcode, but also names, dates, etc.)

If the `layout` argument is omitted, Mork will search for a file named `layout.yml` and load it. If such file cannot be found, Mork will fall back to a default, boilerplate layout (incidentally, this is the layout shown above).

## Scoring response sheets with `SheetOMR`

Assuming that a person has filled out a response sheet by darkening with a pen the selected choices, and that the sheet has been acquired as an image file, response scoring is performed by the `Mork::SheetOMR` class. Three pieces of information must be provided to the object constructor:

- **path**: mandatory path/filename of the bitmap image (accepts JPG, JPEG, PNG, PDF extensions; a resolution of 150-200 dpi is usually more than sufficient to obtain accurate readings)
- **choices**: a named argument (ruby-2 style) equivalent to the `choices` array of integers passed to the `SheetPDF` constructor as part of the `content` parameter (see above). If omitted, the `choices` parameter is inferred from the layout
- **layout_file**: same as for the `SheetPDF` class

The following code shows how to create and analyze a SheetOMR based on a bitmap file named `image.jpg`:

```ruby
# instantiating the object
s = SheetOMR.new 'image.jpg', choices: [5]*100, layout_file: 'layout.yml'
# detecting darkened choice cells for the 100 items
chosen = s.marked_choices
```

If all goes well, the `chosen` array will contain 100 sub-arrays, each containing the list of marked choices for that item, where the first cell is indicated by a 0, the second by a 1, etc. It is also possible to show the scoring graphically by applying an overlay on top of the scanned image.

```ruby
s = SheetOMR.new 'image.jpg', choices: [5]*100, layout_file: 'layout.yml'
s.overlay :check, :marked
s.save 'marked_choices.jpg'
system 'open marked_choices.jpg' # this works in OSX
```

More than one overlay can be applied on a sheet. For example, in addition to checking the marked choice cells, the expected choices can be outlined to show correct vs. wrong responses:

```ruby
...
correct = [[3], [0], [2], [1], [2]] # and so on...
s.overlay :check, :marked
s.overlay :outline, correct
...
```


Scoring can only be performed if the sheet gets properly registered, which in turn depends on the quality of the scanned image.

### Improving sheet registration and marking

For Mork to be able to register the response sheet, the acquired image should be of _sufficient_ resolution and contrast, and it should be straight, with some white margin around the 4 registration circles.

Mork tries to be tolerant of variations in the above parameters, but you should experiment with your own layout, actual printout, and actual scans.

Check for object “validity” to make sure that registration succeeded:

```ruby
s = SheetOMR.new 'image.jpg', choices: [5]*100, layout_file: 'layout.yml'
s.valid?
```

When registration fails, it is possible to get some information by displaying the status and by applying a dedicated overlay on the original image:

```ruby
s = SheetOMR.new 'image.jpg', choices: [5]*100, layout_file: 'layout.yml'
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

#### Wait, why Mork?

Because I used to have a crush on Mindy
