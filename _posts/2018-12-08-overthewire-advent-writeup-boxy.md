---
layout: post
title: "OverTheWire Advent writeup — Boxy"
date: 2018-12-08 06:15:39 +0000
categories: blog
---

### OverTheWire Advent writeup — Boxy

This is a very interesting challenge of reversing. It turns out to be more than just reversing, it requires some knowledge of image processing too, as we will see.

First, once you read into the challenge, the boxy.txt contains many hints for this challenge. Although it implies it is Chinese, as a Chinese native speaker, I can tell you it is certainly not in Chinese. It looks more like Japanese to me. Anyway, I can’t take any advantage from my Chinese knowledge.

I have no clue what boxy.txt tries to hint, so I decided to look into the examples it provides. They are pretty informative and useful.

From the file names, we can figure out those 0xXX.bin has strong relations with those 0xXX.bin*.png files. We need to figure what their relations are. But before that, what are those binary files anyway? Let’s look into them with hexdump -C, like this:

```
hexdump -C
```

```
$ hexdump -C examples/0x00.bin00000000  01 ff 02 ff 03 00 ff 00                       |........|00000008
```

Only 8 bytes. A quick and wild guess would be 0x01, 0x02 and 0x03 here number each field which ends with 0xff, the last 0x00 means the end of record. Not a bad guess at all, right? :)

Well, looking into 0x01.bin clearly indicate my guess is wrong. We need to rethink about 0x00.bin. Wait. Aren’t those bytes mentioned in boxy.txt?

```
### 0x00
```

```
姨嘟 傭 傈媉
```

```
0x01 0xff0x02 0xff0x03 0x010xff 0x00 ; 侸喽 军廎 十呿 侙壌 1
```

Yeah, boxy.txt is to kinda explain all of these binaries with some comments we don’t understand. Therefore, we still have to figure out what it is telling us.

The first thing we can find out is, the last two bytes are always 0xff 0x00, it looks like an end of file or record. If we look at the section 0x01, it contains multiple 0xff 0x00, so it is certainly not an end of file. Does it have anything to do with the PNG files?

0x00.bin has one PNG file related, 0x01.bin has 8 PNG files related, 0x02.bin has 4. These numbers match the number of “0xff 0x00” sequences in each of the binary! So perhaps it means each of block represents each PNG file, and the end of a block is “0xff 0x00”!

How does each block describe each PNG? What does the rest in a block mean? Let’s continue to guess, perhaps it describe each rectangle and each color?

Let’s look at those 0x01* PNG files. Their only differences are colors.

```
### 0x01
```

```
凥妏 力帎
```

```
0x01 0xff0x02 0xff0x03 0x000xff 0x00 ; 巫乸 墰扉 奓夗 嶛儗 00x03 0x010xff 0x00 ; 媑寤 串勸 嬂囎 嫒厏 10x03 0x020xff 0x00 ; 亙慞 乹扚 慨岊 徿彃 20x03 0x030xff 0x00 ; 劳啈 仜姈 抻人 婚处 30x03 0x040xff 0x00 ; 扒廊 傋打 徬壸 崍应 40x03 0x050xff 0x00 ; 届嬎 怺壙 徾屻 佡垫 50x03 0x060xff 0x00 ; 尗彟 嫣拮 嫡峒 偢嫚 60x03 0x070xff 0x00 ; 奥嫄 剁峓 垔佴 嘔偬 7
```

Clearly, we can infer that 0x03 means coloring, while 0x00~0x07 represent 7 difference colors. So, each pair of “0x03 0x..” means coloring the picture with a specified color. With the order, we can map the color to the hexadecimal value. Excellent! But if 0x03 just means coloring, what about the size of the rectangle? And, what does 0x01 and 0x02 in the first two lines mean?

We need to move on to 0x02.bin as its related PNG files have rectangles of different sizes.

```
0x01 0x800x02 0x800x03 0x000xff 0x00 ; 侴当 屳亪 0|0 俸徙 佳姅 仁拴 00x04 0x800x05 0x800xff 0x00 ; 嬻庇 句崁 1|1 互嚒 廉廙 嵌壻 唝哠0x03 0x010x04 0x000x05 0x800xff 0x00 ; 嗉决 塟弸 0|1 忆噑 峭宴 廗习 10x04 0x800x05 0x000xff 0x00 ; 巸俴 偄廳 1|0 岰凥 唔妨 崏坾 惰兰
```

I highlight the parts we need to focus on. We already know 0x03 0x01 means coloring with black. In the last two 0x02* PNG files, there are two black rectangles added, one at a time. Their difference is clear the location, so 0x04 and 0x05 probably mean the location of the rectangle! Likely they are the X-axis and Y-axis.

What’s more, we can also figure out the drawing each PNG file seems incremental, which means we add something on top of the current PNG. When we draw the second black rectangle, we only change its position, so its color remains black once we execute “0x03 0x01”. Now it is clear that there is some global settings of the drawing, something like current color, current X-axis and Y-axis. The binary pairs are instructions to change these values.

From the first two pairs in 0x02 section, it is safe to guess 0x01 and 0x02 probably mean to set sizes, as the rectangles we draw in 0x02* PNG files are just 1/4 of the whole picture. They must represent width and height.

Now, look at what we already figure out:

```
0x01 width or height0x02 height or width0x03 coloring0x04 X or Y axis0x05 Y or X axis0xff draw it
```

```
(For colors: 0x00 means white, 0x01 means black, etc.)
```

For the rest instructions, we have to look into 0x03 binary. Observe each small rectangle drew in each 0x03* PNG file, the drawing location moves from left to right, then from bottom up, then from right to left. It does not need much time to figure out 0x06, 0x07, 0x08 and 0x09 mean relative movement.

Finally, we can find out the meaning of all these instructions:

```
0x01 width0x02 height0x03 coloring0x04 Y axis0x05 X axis0x06 move up0x07 move down0x08 move left0x09 move right0xff draw it
```

```
(For colors: 0x00 means white, 0x01 means black, etc.)
```

And basically we need to interpret these binary commands and execute them to draw pictures, hopefully the flag is hidden in the output PNG files!

With the help of Python Pillow library, it is not hard to come up with the following code:

```
class DrawPNG:  color_map = { 0x00 : 'white', 0x01: 'black', 0x02 : 'red', 0x03: 'green', 0x04 : 'yellow', 0x05: 'blue', 0x06 : 'pink', 0x07: 'cyan' }  num_files = 0  def __init__(self, file_name, color=0x00, x=0, y=0):    self.color = color    self.x = x    self.y = y    self.file_name  = file_name    self.im = Image.new('RGBA', (256, 256), self.color_map[self.color])    self.instructions = { 0x03 : self.change_color, 0x01 : self.set_width, 0x02 : self.set_height, 0x04 : self.set_y, 0x05 : self.set_x, 0x07 : self.move_down, 0x06 : self.move_up, 0x09 : self.move_right, 0x08 : self.move_left, 0xff : self.draw_it }  def change_color(self, val):    self.color = val  def set_width(self, val):    self.width = val  def set_height(self, val):    self.height = val  def set_x(self, val):    self.x = val  def set_y(self, val):    self.y = val  def move_up(self, val):    self.y -= val  def move_down(self, val):    self.y += val  def move_left(self, val):    self.x -= val  def move_right(self, val):    self.x += val  def draw_it(self, unused):    draw = ImageDraw.Draw(self.im)    print "x = %d y = %d w = %d h = %d"%(self.x, self.y, self. width, self.height)    draw.rectangle((self.x, self.y, self.x + self.width, self.y + self.height), fill=self.color_map[self.color])    self.im.save(self.file_name + '_' + str(self.num_files) + '.png')    self.num_files += 1  def parse_instructions(self, data):    for i in xrange(0, len(data), 2):      print "%x : %x"%(data[i], data[i+1])      self.instructions[data[i]](data[i+1])    return self.num_files
```

It works, but the output is not what I expect. We get some apparently morse code in 913 PNG files! Like this one:

So the flag must be hidden in the morse code! But the first question is how do we extract it?? Clearly we don’t want to read all 913 files and type them in a morse code translator!

My first thought is to use OCR, which can easily to extract texts from such a simple PNG file. Unfortunately the following pyocr code I tried doesn’t recognize morse code at all!

```
tools = pyocr.get_available_tools()[0]
```

```
text = tools.image_to_string(Image.open("reverseme_12.png"), lang="eng", builder=pyocr.builders.TextBuilder(tesseract_layout=6))
```

Perhaps it doesn’t know whether to recognize the line as ‘-’ or ‘_’. Shrug. I give up. How about extracting it with our own code by reading each pixel? It sounds hard, but in this case it is actually easy, because all the dots have the same size, all the lines too, more importantly, they are all in the middle line of each picture! This means we don’t need have to read every pixel, we just need to read the pixels along the middle line!!!

```
def chunkstring(string, length):    return (string[0+i:length+i] for i in range(0, len(string), length))
```

```
# Read pixels on the middle line# Return empty string is the PNG is blank# Else, return the morse code stringdef parse_morse(file_name):  im = Image.open(file_name)  pix = im.load()  total_dots = dots = 0  result = ""  for x in range(256):    if pix[x,128] == ImageColor.getcolor('Black', 'RGBA'):      total_dots += 1      dots += 1    else:      if dots > 5:        result += '-'      elif dots > 0:        result += '.'      dots = 0  return ' '.join(list(chunkstring(result, 5)))
```

Of course, because each picture is drew incrementally, only the final picture has a complete morse code. We can check this by checking if there is a blank picture following it. The rest of the code is even easier:

```
file_name = sys.argv[1]file_len = os.path.getsize(file_name)data = array('B')with open(file_name, 'rb') as f:    data.fromfile(f, file_len)f.close()
```

```
draw = DrawPNG(file_name)num_files = draw.parse_instructions(data)
```

```
has_complete_morse = Falsenums = []for i in xrange(num_files-1, 0, -1):    png_file = file_name + '_' + str(i) + '.png'    morse_code = parse_morse(png_file)    if len(morse_code) ==0 :      print "%s is blank!"%(png_file)      has_complete_morse = True    elif has_complete_morse:      print "%s has complete morse code %s"%(png_file, morse_code)      plain_text = decode_morse(morse_code)      print "%s has text %s"%(png_file, plain_text)      has_complete_morse = False      nums.append(chr(int(plain_text)))
```

```
nums.reverse()print ''.join(nums)
```