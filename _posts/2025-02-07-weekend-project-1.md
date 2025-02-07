---
layout: post
title: "One week(end) project - Note taking app"
author: "Michał Kozłowski"
tags: "Weekend project"
image: /assets/images/weekend-project-1/1.png
excerpt_separator: <!--more-->
---

I'm frustrated by the state of note taking apps with stylus support for computers. There are a couple of options but not a single one is perfect for my use case. So I created my own. <!--more-->

# Perfect note taking app
In my search for the perfect stylus note-taking app, I came across many options. Here are, in my opinion, the three best ones:
1. One Note
2. Excalidraw
3. Xournal++

But as you can probably tell from the intro I've encountered problems with every one of them. Excalidraw is browser based and its stylus drawing feels unresponsive. Xournal doesn't support vectors (which are important for good PDF export), and its drawing space is limited to A4 aspect ratio. This left me using One Note. However there's one major problem, that all of those apps have, they don't support Markdown. While I take a lot of notes by hand, I also type a lot, and there doesn't seem to be an app combining both good typing experience and responsive stylus on infinite canvas. So here is what in my opinion would make a perfect note-taking app.

1. Markdown support - like Obsidian
2. responsive drawing experience with vectors
3. infinite canvas
4. importing Images and PDFs
5. export functionality

I also tried Excalidraw plug-in for Obsidian but I didn't like it. I want to have drawings and text in the same place.

# How to create an app?
The thing is I'm totally new to creating apps with user interface. I know I won't be able to create polished product, but I want to try something new. To not commit myself to much to this project I gave myself a weekend. I started by consulting with our friend GPT and it proposed using [Qt](https://www.qt.io/download-open-source) framework. It is used for creating apps that can run on different platforms, and is open source (for open source use case). If you want to start working with `Qt` simply install community edition. This was a great pick, everything from text boxes, vector paths and even a tablet stylus support, I had every puzzle piece. Qt has a great library of example projects that you can build and run with a couple of clicks. And one of them was drawing app with stylus. 

![]({{ "assets/images/weekend-project-1/2.png" | relative_url }})

I kept very little from this example, but seeing something work right away was encouraging. 

## Drawing lines
I created new class in separate file, and started by defining a vector stroke. It holds A `QPen object` for color and width, and a `vector` of points.

```cpp
struct Stroke {
    QPen pen;
    std::vector<QPointF> points;
};
```
Then I implemented canvas from the example but simplified it as much as I could. You can see `TabletCanvas` holds a list of `Stroke`s, a pointer to the `Stroke` that is currently being painted and a `QPen` object.
```cpp
class TabletCanvas : public QWidget {
    Q_OBJECT

public:
    explicit TabletCanvas(QWidget *parent = nullptr);

protected:
    void tabletEvent(QTabletEvent *event) override;
    void paintEvent(QPaintEvent *event) override;
    void resizeEvent(QResizeEvent *event) override;

private:
    qreal pressureToWidth(qreal pressure);
    void startStroke(const QTabletEvent *event);

	bool m_drawing;
    std::vector<Stroke *> m_strokes;
    Stroke *m_currentStroke;
    QPen m_pen;
};
```
I also override the methods from `QWidget` that being `tabletEvent` which runs every time something happens to a tablet, for example stylus moving next to it. `paintEvent` which will run every time we draw something onto the canvas. And finally, the `resizeEvent`, which will run (you guessed it) every time the main window is resized. I also added helper methods `pressureToWidth` and `startStroke`. Before I'll show you how I handled `tabletEvent`, we need to tell our object to listen for said events, we can do it in the constructor using `setAttribiute`.

```cpp
TabletCanvas::TabletCanvas(QWidget *parent)
    : QWidget(parent),
    m_pen(Qt::white, 1.0, Qt::SolidLine, Qt::RoundCap, Qt::RoundJoin),
    m_currentStroke(nullptr),
    m_drawing(false)
{
    resize(500, 500);
    setAttribute(Qt::WA_TabletTracking);
}
```

Now that our canvas will listen to specific tablet related events I can show you my very straight-forward implementation of `tabletEvent`.

```cpp
void TabletCanvas::tabletEvent(QTabletEvent *event)
{
    switch (event->type()) {
    case QEvent::TabletPress: {
        // User pressed the tablet start a stroke
        break;
    }
    case QEvent::TabletMove: {
	    // User is moving the stylus 
        if (m_drawing) {
            // And is pressing onto tablet
            // update the stroke based on current position
        }
        break;
    }
    case QEvent::TabletRelease: {
        // User stopped pressing the tablet stop drawing
        break;
    }
    default:
        break;
    }
    event->accept();
}
```

I think this is self-explanatory lets see how to start a stroke. To my helper method `startStroke` I passed a pointer to the event that initiated it. This is because we need the position as well as the pressure of that event.

```cpp
void TabletCanvas::startStroke(const QTabletEvent *event)
{
    m_drawing = true;
    Stroke *stroke = new Stroke;
    stroke->pen = m_pen;
    qreal initialWidth = pressureToWidth(event->pressure());

    stroke->pen.setWidthF(initialWidth);
    stroke->points.push_back(event->position());
    m_strokes.push_back(stroke);
    m_currentStroke = m_strokes.back();
}

qreal TabletCanvas::pressureToWidth(qreal pressure)
{
    return pressure * 5 + 1;
}
```

First I set `m_drawing` flag to `true` to know that I should add points when stylus is being moved (see `tabletEvent` from before). Then I create new `Stroke` object and save its pointer to `stroke`. Some pen updates later I `push_back` the `stroke` pointer to `m_strokes` and update `m_currentStroke` to `stroke`. Now when we detect that stylus moved we can just continue to add new points to the `m_currentStroke` resulting in a line. Lets see our updated `tabletEvent` method.

```cpp
void TabletCanvas::tabletEvent(QTabletEvent *event)
{
    switch (event->type()) {
    case QEvent::TabletPress: {
        startStroke(event);
        break;
    }
    case QEvent::TabletMove: {
        if (m_drawing) {
            m_currentStroke->points.push_back(event->position());
            update();
        }
        break;
    }
    case QEvent::TabletRelease: {
        m_drawing = false;
        m_currentStroke = nullptr;
        break;
    }
    default:
        break;
    }
    event->accept();
}
```

We can run our app but the displaying part is incomplete. To finish up the drawing we need `paintEvent` method. The most straight-forward method would be to update the lines every time something changed. This can be achieved by looping over lines in `m_strokes` and connecting each point with a `QPainterPath`.

```cpp
void TabletCanvas::paintEvent(QPaintEvent *event)
{
    QPainter painter(this);
    painter.setRenderHint(QPainter::Antialiasing);
    for (const Stroke *stroke : m_strokes) {
        if (stroke->points.empty())
            continue;
        painter.setPen(stroke->pen);
        // If the stroke only has one point, draw a point.
        if (stroke->points.size() == 1) {
            painter.drawPoint(stroke->points[0]);
        } else {
            // Create a QPainterPath and connect all points.
            QPainterPath path;
            path.moveTo(stroke->points[0]);
            for (size_t i = 1; i < stroke->points.size(); ++i) {
                path.lineTo(stroke->points[i]);
            }
            painter.drawPath(path);
        }
    }
}
```

After setting up our main app to create canvas and linking it to base class. we can see the results.

![]({{ "assets/images/weekend-project-1/3.gif" | relative_url }})

But this implementation of `paintEvent` is poorly optimized. When moving the cursor we constantly run `paintEvent`, resulting in avg. of 264 executions/second. Given that our update algorithm is O(n^2), this will quickly become to much for the app. Best fix for this is to cache our previously rendered lines to a texture. Look at this code:

```cpp
void TabletCanvas::updateCache(){
    QPainter painter(m_canvasCached);
    painter.setRenderHint(QPainter::Antialiasing);
    for (const Stroke *stroke : m_strokes) {
        if (stroke->points.empty())
            continue;
        painter.setPen(stroke->pen);
        // If the stroke only has one point, draw a point.
        if (stroke->points.size() == 1) {
            painter.drawPoint(stroke->points[0]);
        } else {
            // Create a QPainterPath and connect all points.
            QPainterPath path;
            path.moveTo(stroke->points[0]);
            for (size_t i = 1; i < stroke->points.size(); ++i) {
                path.lineTo(stroke->points[i]);
            }
            painter.drawPath(path);
        }
    }
}
```

I only moved the code to a helper function and changed the argument when creating a `QPainter` object. I now pass `m_canvasCached` (pointer to a`QPixmap`) instead of `this`. By moving the code to different method we can now edit the `paintEvent`.

```cpp
void TabletCanvas::paintEvent(QPaintEvent *event)
{
    Q_UNUSED(event);
    QPainter painter(this);

    // Draw cached strokes
    painter.drawPixmap(0, 0, *m_canvasCached);

    // Draw new stroke
    if(m_drawing && m_currentStroke)
    {
        painter.setPen(m_currentStroke->pen);
        QPainterPath path;
        path.moveTo(m_currentStroke->points[0]);
        for(size_t i = 1; i < m_currentStroke->points.size(); i++)
        {
            path.lineTo(m_currentStroke->points[i]);
        }
        painter.drawPath(path);
    }
}
```

Only other change needed is to add `updateCache()` to run on `TabletRelease` event in `tabletEvent` method:

```cpp
. . .

case QEvent::TabletRelease: {
        m_drawing = false;
        m_currentStroke = nullptr;
        updateCache();
        break;
    }
    
. . .
```

Now the `paintEvent` method is O(n) for the number of points in a current stroke. By making this change we run the expensive O(n^2) method only once a stroke. This stops the app from lagging under heavy usage, and also saves some processor usage:  

![]({{ "assets/images/weekend-project-1/4.gif" | relative_url }})
Not optimized version

![]({{ "assets/images/weekend-project-1/5.gif" | relative_url }})
Optimized version

This optimization is of course not free. You can see that the memory used by app goes up. Another changes that could be made for further optimization:
1. limiting number of updates/s 264 is excessive.
2. updating only the part of the texture that changed, not the whole canvas
3. maybe also caching current stroke?

But for my case that would be an overkill.

## Text?
Yeah so by the time I got the drawing working the weekend was long gone. This was probably to be expected, I have never worked with `Qt` and everything was new to me.  This led to the text feeling more like a `Serving suggestion` then a polished feature. I’m not particularly proud of how it turned out, so let’s just move on.
I wanted the text to be separated in blocks by titles like this

![]({{ "assets/images/weekend-project-1/6.png" | relative_url }})

I ended up creating a derived class from `QTextEdit`. I wanted the text box to scale infinitely when user is typing. I didn't find a way to do it properly, so I just check if user is typing, and then re-scale the box accordingly to a font size. To achieve this I used `textChanged` signal from `QTextEdit`. In a constructor I connected the event with my `resizeText` method

```cpp
connect(this, &CustomText::textChanged, this, &CustomText::resizeText);
```

And implemented buggy resize functionality
```cpp
void CustomText::resizeText()
{
    QString text = toPlainText();
    if (text.isNull() || text.isEmpty()) {
        setFixedWidth(100);
        setFixedHeight(30);
        return;
    }
    const QFontMetrics &font = fontMetrics();
    QSize textSize = font.size(Qt::TextShowMnemonic, text);
    int width = textSize.width() + padding;
    int height = textSize.height() + padding / 2;
    if (width < 100) {
        width = 100;
    }
    if (height < 30) {
        height = 30;
    }
    setFixedHeight(height);
    setFixedWidth(width);
}
```
which by the way doesn't work with `tab` key, *ehhh*.

Then I created second class that holds the `vector` of `CustomText`. That could probably be implemented with less abstraction, but I didn't really care at the time. I also use layout feature from `Qt` to align the boxes vertically based on their size. 
```cpp
TextBlock::TextBlock(const QPoint &pos, QWidget *parent)
    : QWidget(parent), position(pos)
{
    move(pos);

    QVBoxLayout *layout = new QVBoxLayout(this);
    layout->setContentsMargins(0, 0, 0, 0);
    layout->setSpacing(0);
    layout->setSizeConstraint(QLayout::SetMinimumSize);
    setLayout(layout);

    CustomText *ct = createCustomText(1);
    layout->addWidget(ct);

    adjustSize();
    show();
}

```
Then simple creator method for `CustomText` which also connects the handling of `Enter` and `Backspace` keys
```cpp
CustomText* TextBlock::createCustomText(bool firstOne)
{

    CustomText *ct = new CustomText(!firstOne, this);
    ct->setSizePolicy(QSizePolicy::Preferred, QSizePolicy::Minimum);
    // Allow the widget to accept focus when clicked.
    ct->setFocusPolicy(Qt::ClickFocus);
    ct->setReadOnly(false);

    // Connect the enterPressed signal so this widget notifies TextBlock.
    connect(ct, &CustomText::enterPressed, this, &TextBlock::onCustomTextEnterPressed);

    connect(ct, &CustomText::backPressed, this, &TextBlock::onCustomTextBackPressed);

    customTexts.append(ct);
    return ct;
}
```
About the signals, when user presses `Enter` i check if the current line starts with `"# "` (marking a title). If so I detach the text to a new `CustomText` and create next line.
```cpp
void TextBlock::onCustomTextEnterPressed(CustomText *sender)
{
    // Get the full text entered by the user.
    QString senderText = sender->toPlainText();
    if (senderText.isEmpty())
        return;

    QStringList lines = senderText.split("\n", Qt::SkipEmptyParts);

    bool titleFound = false;

    QString lastLine = lines[lines.count()-1];
    if (lastLine.trimmed().startsWith("# ")) {
        CustomText *block = createCustomText(); // Create new block of text
        layout()->addWidget(block);
        if(lines.count() > 1) // If title is not firs line
        {
            block->setPlainText(lastLine); // We create one block more
            lines.pop_back();
            CustomText *block2 = createCustomText();
            layout()->addWidget(block2);
        }
        QString newText = lines.join("\n");
        sender->setPlainText(newText);

    }else
    {
        sender->append("");
    }
}
```
See this in action:

![]({{ "assets/images/weekend-project-1/7.gif" | relative_url }})

![]({{ "assets/images/weekend-project-1/8.gif" | relative_url }})
Only UI button that have a real function

# Full code
If you want to see full code go to [https://github.com/M1chol/m1-notes](https://github.com/M1chol/m1-notes)

## My thoughts
In the end the whole project took about 4 days, which is 2 days more then planed. This whole format was very spontaneous, and I plan to try it again sometimes. I really liked working with `Qt` and I'll revisit it sometime. When it comes to the end product, I came nowhere near meeting my own expectations.