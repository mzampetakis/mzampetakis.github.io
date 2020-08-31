---
title: AquaGo - A Toy Project
description: A Digital Aquarium written in pure golang!
image: /assets/posts/Aquago-A-Toy-Project/auqago.png
show_image_post: true
date: 2020-08-31 22:40:00 +0300
published: false
categories: []
tags: [toy-project,golang,coding,oAuth,image-processing,LEGO]
---

Just imagine if you could have on your living's room TV an aquarium! Wouldn't that be nice, especially during the hot period of summertime? What about if you could add digital fishes and decorative stuff in this aquarium that you could create (physically or by design) with you own style?

This is more or less what I got up with during this summer! 
> A real toy project!

Two beloved things helped me implement this project. The first one is the Go programming language (alongside [ebiten](https://ebiten.org/) a dead simple 2D gaming library for GO) and LEGOs which I used to decorate and give "life" to my digital aquarium.

I hope you enjoy reading this as much I enjoyed working on this and you might want to give it a try!  

## Inspiration
While watching a video of a [tour in the LEGO house in Denmark](https://youtu.be/Siy1F7fUiY0?t=501) I came across a really interesting (among a lot of) part! At 08:21 a visitor of the LEGO house can create with a few LEGO bricks a structure (sometimes they look like a fish!) that could be added "inside" a digital aquarium after a scanning process.

That digital aquarium is consisted of a static background, some seaweeds made of LEGO bricks that wave, some background decorative stuff and the "fishes" that visitors make out of LEGOs after adding some facial stuff (mouth, eyes etc). The scanning process of each fish takes place in a special illuminated pad that visitors put their LEGO "fish" and a camera captures it.

This is what gave inspiration to design and develop this project during the summer of 2020. 

> I really enjoyed the process of designing, developing and using it as well!
 
I propose you to watch the whole video as it is really interesting and maybe you could find something to play with. 😉

## Setting Requirements

### Language 💻
Before starting implementing this, the idea was wondering around my mind for a few days. It was clear that I would need some image processing, and a gaming (2D) library. Go was already chosen as it is something I am learning at this period of time and would like to improve my skill at it somehow. [Ebiten](https://ebiten.org/) supported my choice, as Go is not a graphics/game oriented language and I had some concerns on using this at this project. 

## Assets 🐠
So for giving life to my digital aquarium I would require to have a few things inside it to make it look alive! First of all I would like some decorative objects that could be moved (draggable) inside the aquarium. These objects are usually some seaweeds, some rocks or even a treasure chest! I would also like to have some bubbles which would wobble towards the surface and have no other purpose rather that making te aquarium looking more realistic (as it couldn't)! The last and more complex part are the "fishes". Fishes would move "freely"inside the aquarium to different directions, with different speed (or just stopped) and change moving angle! Of course we would like to see them waving as they move around!  

## Image Transformation 🏞
In the image transformation part, it is clear that there are lots of different libraries out there and of course there is always the [OpenCV](https://opencv.org/) project which could be there for me if needed. So image processing was not something to wonder about. The images should need some sort of transformation. Resizing and cropping would be the simpler tasks. But among them it would be pretty cool if the application could remove any static background and make it transparent giving a more realistic result to our assets.

### No Interaction 🙌
The last requirement was to be able to use it with less interaction as possible. I didn't want to rerun or rebuild my app when I wanted to add a new element (decorative or fish) in the aquarium. For this reason I wanted an online hosting where I could put my images (capture by a smart phone!) and the application could download them afterwords and put those stuff inside the aquarium. Isn't that nice?

> That's all. Nothing more - nothing less (at least in the beginning)!

## Development
🎲🎲 
It seems reasonable that, as real life seems to move a little bit randomly, a random number generator function should be extensively used at initialization and runtime in order to create a more realistic and fuzzy result.  
🎲🎲
>So prepare to get surprised

All necessary parameters that could be changed to meet someones needs are saved as env vars or within a special file '.env'. More details for this can be found in the [project's README](https://github.com/mzampetakis/aquago#env-vars-).

### Tools / Libraries
Apart from Go language and the [Ebiten library](https://ebiten.org/) that used for the visualization ad movement of the aquarium's assets, the [bild library](https://pkg.go.dev/mod/github.com/anthonynsimon/bild) used as a collection of parallel image processing algorithms.

For the online storage and retrieval mechanism for te assets, [Google Drive]( https://drive.google.com) used and exploited through the [Google API v3](https://developers.google.com/drive/api/v3/about-sdk) . To make those tasks easier a [Google Drive's API wrapper library]( https://github.com/mzampetakis/gogle-drive ) developed and used. The library supports the auth/authz part as well as general methods to search for files (using some filter criteria) and to download specific files. Exactly what we needed for our case!

### Bootstrapping 
So what happens when the application starts. First of all it tries to retrieve all the newly added assets from the remote Google's Drive folders. This is done through a goroutine which is initiated to check every minute for newly added images and download if found from the Google Drive. The mechanism to do this scheduling was implemented this way:

```go
ticker := time.NewTicker(1 * time.Minute)
go func() {
    for range ticker.C {
        downloadNewImages(gogledrive, os.Getenv("BGFOLDERID"), gdriveBgFolder, bgFolder)
        downloadNewImages(gogledrive, os.Getenv("FGFOLDERID"), gdriveFgFolder, fgFolder)
    }
}()
```

Using this way we will see our assets to pop up into the aquarium gradually.

For each new image a background separation and removal algorithm removes the background based on the fact that is is a single solid color. For this reason all images are captured with a solid color background(generally white but it could be any other color)!

This is the image transformation that occurs to each retrieved asset:

![Image Transformation process](/assets/posts/Aquago-A-Toy-Project/imgtransform.png)

Then, all transformed images/assets are placed within the aquarium with some specific and some random initial values as described in the following sections.

### Bubbles
From the first frame and (about) every 1" one new bubble pops in from the bottom of the aquarium and starts to move to the top. Each bubble differs in size and speed and moves upwards and also to a little bit to the left or to the right randomly. In fact it moves to one direction and based on a threshold it can randomly change direction.

```go
randDirection := Direction(rand.Intn(2))
if randDirection == 0 {
    randDirection = -1
}
scale := Scale(rand.Float64() + 0.5)
b := &Bubble{
    image:     bblImage,        //we could use different images in the future but now we use a single one
    x:         100,             //leave a standard margin from the left
    y:         screenHeight,    //bottom of the image
    direction: randDirection,   //random dinitial irection - left/right
    speed:     Speed(rand.Intn(int(maxSpeed+1)) + 1), //random speed
    scale:     scale,   //random scale
}
```
Using this loop all bubbles were updated with a frequency based on their speed! Bubbles could disapear randomly or after reaching the top of the screen and direction could randomly change!

```go
for idx, b := range g.bubbles {
    //Determines speed by changing bubble's position according to frame count
    if frameCount%int(b.speed*-1+(maxSpeed+2)) == 0 {
        _, h := b.image.Size()
        //remove if bbl reaches top or randomly
        if b.y <= -h || rand.Float64() < bblDisappearPossibility {
            g.bubbles = append(g.bubbles[:idx], g.bubbles[idx+1:]...)
            bblMux.Unlock()
            continue
        }
        b.y--
        // 33% chance to go just straight up
        // looks more smooth in transition
        if rand.Intn(3) == 1 {
            if rand.Float64() < bblDirectionPossibility {
                b.direction *= -1
            }
            b.x += int(b.direction)
        }
    }
}
```

All this resulted in this beautiful result:

![Moving Bubbles](/assets/posts/Aquago-A-Toy-Project/bubbles.gif)

### Background Items
Background items are retrieved through Google Drive at bootstrap and are checked for newly added every minute. If any new asset is found it is processed and then randomly is being placed into the aquarium. These items are able to be repositioned by dragging them so we could decorate the aquarium based on our mood!

> These items are a bit dull as their only option is the image used and their position.

![BG assets](/assets/posts/Aquago-A-Toy-Project/bg.png)

### Foreground Items
Time to give some life to our aquarium! This one was the biggest challenge! Digital fishes should move around the aquarium to different directions with different speeds and angles and all of them could be randomly change - something as real fish in an aquarium! So, as you can see, the fish structure holds oll those relevant data alongside some basic information for current position and it's name (that was the file's name)!


```go
w, h := fishImage.Size()
randDirection := Direction(rand.Intn(2))
if randDirection == 0 {
    randDirection = -1
}
angleDirection := rand.Intn(2)
if angleDirection == 0 {
    angleDirection = -1
}
f := &Fish{
    image:         fishImage,
    name:          d.Name(),
    x:             rand.Intn(screenWidth - w),
    y:             rand.Intn(screenHeight - h),
    direction:     randDirection,
    speed:         Speed(rand.Intn(int(maxSpeed+1)) + 1),
    angle:         rand.Float64() / 2 * float64(angleDirection),
    skew:          float64(0.05),
    skewDirection: 1,
}
```

Angle used to allow fishes move upwards and downwards while skew and skew direction used to transform each fish in order to see them 'swimming' - somehow. Skew changes direction more frequently when fishes move faster compare to the slower one.

This is the part that updates our fishes in the aquarium:

```go
for _, f := range g.fishes {
    //Determines speed by changing fishe's position according to frame count
    if frameCount%int(f.speed*-1+(maxSpeed+2)) == 0 {
        w, h := f.image.Size()
        if f.y <= 0 || f.y >= screenHeight-h {
            f.angle *= -1
        } else {
            if rand.Float64() < fishAnglePossibility {
                angleDirection := rand.Intn(2)
                if angleDirection == 0 {
                    angleDirection = -1
                }
                f.angle += (rand.Float64() - 0.5) / 2
            }
        }
        changeSpeedRand := rand.Float64()
        if changeSpeedRand < 0.05 && f.speed > 1 {
            f.speed--
        }
        if changeSpeedRand > 0.95 && f.speed <= maxSpeed {
            f.speed++
        }
        if f.speed > 0 {
            //change direction if close to left/right or randomly
            if f.x <= 0 || f.x >= screenWidth-w || rand.Float64() < fishDirectionPossibility {
                f.direction = Direction(int(f.direction) * -1)
            }
            f.x += int(f.direction)

            f.y += int(f.angle * 3)
            if frameCount%(int(f.speed*-1+(maxSpeed+2))) == 0 {
                if f.skew >= 0.2 || f.skew <= -0.2 {
                    f.skewDirection *= -1
                }
                f.skew += float64(f.skewDirection) * 0.01
            }
        }
    }
}
```

## Result
Time for fun 🎆  
Combining all those we have something that meets our requirements:

![Aquago in action](/assets/posts/Aquago-A-Toy-Project/aquago.gif)

## Challenges faced
The most challenging aspect of this project was the part of designing. One basic requirement should be met - be as easy as possible to update aquariums content. This was the reason that google drive set up, background removal algorithm incorporated, polling for new images added and so on...

The next challenging part was to make this app run effortlessly by any user. So easy as grabbing the code, setting up Google Drive's oAuth and run it.

The last one had to do with the gaming part of the app. Now being a game developer made me try harder to get up with how are game engines (such as ebiten) usually work. To be honest, I really enjoyed this part as this introduced me in a new field!

## Improvements
* The first major improvement that has to be done has to do with the background removal algorithm (contour extraction could be used here). There are numerous algorithms to use, however, we have to keep performance in mind which is critical at least at startup time.

* Some tasks could be executed concurrently. For instance each image could be downloaded and then processed asynchronously.

* Also we might consider removing the necessity to use Google Drive in case someone doesn't want to use it.

* Fish waving function could also be improved to provide a more realistic result.

## More Info
The whole project is hosted under [https://github.com/mzampetakis/aquago](https://github.com/mzampetakis/aquago) repo where you can find more technical details of the project.The Google's drive wrapper library is located under [https://github.com/mzampetakis/gogle-drive](https://github.com/mzampetakis/gogle-drive) repo with some technical details as well.

### Contributions are more than welcome in form of PRs, issues, even with a simple comment down here! But most importantly I would like to know if you enjoyed this!
