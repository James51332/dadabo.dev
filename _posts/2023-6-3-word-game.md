---
layout: post
title: Creating a Word Themed Game
---

![](/assets/img/word-game/title.png)

Recently, I took part in my state's FBLA competition, in the game design category. This year's theme was to make a word inspired game. While my experience in game developmentis limited, I managed to use my programming skills to create what I think, is a fun andchallenging word puzzle game, similar to the likes of Wordle.

<!--more-->

### The Design of the Game

The first challenge I faced was generating an idea for a game that would be reasonably
fun, but also manageable to implement. I ponderered this for several weeks while I put
the project to the side, but eventually settled onto an idea of connecting words. The player
would be given both a starting and a target word, and would connect them by changing one letter
at a time. Each time they changed a letter, however, they still must have a valid word. For example,
a starting word of CAT and target of BOT could be attained by going CAT to BAT and BAT to BOT. This
idea led me to put together several puzzles, which seemed reasonably fun. With the idea in motion,
I used ChatGPT to generate a name for the game. I arrived at Lettermorph. The next step was
to choose a toolset to build with.

I weighed several options. While it would certainly aid me in the long run, I lacked the familiarity
with game engines to make use of them properly. I still do not have a strong grasp on the paradigms
and practices to implement a standard 3D game in an engine such as Unity, much less a more UI based
game. Therefore, I opted to use SDL as the backend for my game, as my comfort in C++ would allow for
much faster progress. I used premake to generate the build system, so that in the future, the code may
be more easily migrated to other platforms (more on this to come).

Next, I planned the design of the game. I used a rectangular design, which can be seen in the title art, 
both because it was easier to render and because it fit the Wordle-esque vibe of the project. I also generated
a simple color palette which could remain consistent throughout the game.

![Four colors used in game](/assets/img/word-game/colors.png)

With all of the preparation and planning done, I finally moved to the implementation of game.

### Mechanics: The Dictionary

The dictionary was initially the most intimidating part of the project. I would need to be able to check
whether a word inputted by the user was a valid english word on the fly. It wasn't immediately clear how
to do this, given how many words there are, and there is not a C++ API or library that can aid in this.
The solution I ended up with uses a large list of words to create a simple API for the game to query.

First, I found a list of around 60,000 sorted words online with a quick search and saved it a text file 
in the resources directory of the game. I used the SDL file API (which works nicely with macOS bundles) 
to load this into a large buffer of characters, which is tokenized and placed into an array. We can capitalize
off of the fact that the list is sorted alphabetically when checking for a given word. Here is an algorithm,
with some implementation details omitted, which implements a binary search to check for any given word.

```cpp
std::vector<std::string> dictionary;

bool isValidWord(const std::string& word)
{    
    // Maintain indices of our possible range.
    std::size_t beginning = 0;
    std::size_t end = dictionary.size() - 1; 
    for (;;)
    {
        // Select the middle word of range.
        std::size_t middle = (beginning + end) / 2;
        std::string dictionaryWord = dictionary[middle];

        if (wordsEqual(word, dictionaryWord)) return true; 

        // If we've narrowed to one element, we're done.
        if (beginning >= end) return false;

        // Narrow our range of where the word maybe be.
        if (alphabeticalOrder(dictionaryWord, word)) 
            beginning = middle + 1;
        else
            end = middle - 1;
    }
}
```

We start with the center word, and attempt to eliminate one half of the dictionary from our search. 
Next we move to the quarter, and the eighth, and so on and so forth. This repeats, until our search 
range is small enough that we know the word isn't in the list, or until we find the word. The gives 
us roughly log2(n) steps to check, which is only about 16 steps for a dictionary of 60,000 words. This 
is a totally shippable approach.

### Improving the User Experience

The other core mechanics of the game were not technically interesting. The scene manager is possibly 
an exception.  I implemented a simple scene manager that keeps a stack of the scenes that are presented, 
and updates the one on top. The base scene of the stack is the main menu. Scenes can push on top of 
this to allow more manageable switching between scenes, without additional logic. For example, the 
settings menu may be pushed on the top of the scene stack, so that the back button will return to 
either the main menu or level scene depending on how it was accessed. A hash map is used to allow 
scenes to refer to each other by name. 

Two systems that am more excited to write about are the UI and Animation systems. The UI in the game is pretty basic,
but it is designed using the immediate mode GUI paradigm, and is both easily added to a scene, and can be extended
to add more features in the future. Here is an example of the code to implement some basic UI in the game. Note that
this isn't completely verbose, as the actual API requires slightly more parameters about the sizing of sliders.

```cpp
UI::Text("Music Volume", Renderer::GetWidth() / 2, 100);
if (UI::Slider(Renderer::GetWidth() / 2, 150, musicVolume))
    Mixer::SetMusicVolume(musicVolume);

if (UI::Button("Back", Renderer::GetWidth() / 2, 300))
    SceneStack::PopScene();
```

With this code, the game implements a main menu, as well as several different scenes that interact with 
each other nicely. But how does it work? The UI system is designed to be entirely stateless. Each time 
that the API is called, the system determines whether the mouse is interacting with the element, either 
by hovering, which affects the appearance, or by clicking, which results in behavior specific to each 
element. Here is the code that creates a button for some given text.

```cpp
bool UI::Button(const char* text, float x, float y)
{
    // Calculate the size of the button.
    float width, height;
    UI::ButtonSize(text, &width, &height);

    // Determine if the button is highlighted
    Rect bounds = {x - width / 2, y - height / 2, width, height};
    Vec2 mouse = Input::GetMousePosition();
    bool highlighted = pointInRect(mouse, bounds);

    // Draw the button based on highlight status
    if (highlighted)
        Renderer::Fill(Color::Highlight);
    else
        Renderer::Fill(Color::Dark);

    Renderer::Rect(bounds);
    UI::Text(text, x, y);

    return (highlighted && Input::MousePress(Mouse::Left));
}
```

This same principle is used to implement the sliders. The only minor exception to the rule is that 
for sliders, the UI system maintains the ID of a selected slider, so that even if the mouse is dragged 
off the visual element, the slider will continue to update. IDs are simply assigned in order as sliders 
are drawn per frame, and are not a public part of the API. This is safe practice because it is reasonable 
that the UI within a scene will not change while any slider is in use; however, nothing is done to protect
against this, if there were a case in which such happened.

Here is a demo of the sliders and buttons in the game that are to change the volume and move between 
levels. Note that the clip is zoomed for clarity, and doesn't show the full game.

![User Interface Demonstration](/assets/img/word-game/ui-demo.gif)

The next and final system that I'll discuss is the animation engine. I wanted to create a system that was both
easily used, as well as easily extensible with different types of animations. This boiled down to creating 
a central manager of animations, to which the client can register animations of several types, and query their
status. The API also exposes several other settings to control iterations and other various aspects of animations.

```cpp
enum class AnimationType
{
    Lerp,
    Wave,
    Pulse,
    EaseInOut;
}
    
struct Animation
{
    AnimationType Type;

    bool Active;
    bool ResetOnInactive;
    bool ResetOnComplete;
    bool Loop;

    float Progress;
    float Duration;
    float Max;
    float Min;

    float Value;
};

struct Animator
{
    using ID = std::size_t;

    static ID Register(const Animation& animation);
    static const Animation& Query(ID animID);

    static void Reset(ID animID);
    static void SetActive(ID animID, bool active = true, float delay = 0.0f);
};
```

While the implementation details are not shown, the purpose is the code snippet is to demonstrate the API and its
extensibility. I could easily implement new animation types, which was something that I actually did throughout
development. I used animations to create scrolling letters in levels, pulsing animations of typed characters, and
shake animations.

### The Road to Nationals

To conclude, I want to briefly discuss my ambitions for the national FBLA conference in Atlanta. The most notable
feature that I hope to achieve is crossplatform support. The project should be relatively easy to port to Windows
and, should I choose, Linux machines. However, I think a much more useful platform to support is WASM. I have already
managed to use emscriptem to compile the code for the browswer. The text was relatively limited, and the game will
need to slightly be restructured before this version is deployabale. For one, audio support is slightly different on
web platforms. This is because sites shouldn't spam the user with music and effects as soon as they open the page.
This fix should be minor, though. The other change is the to UI scale. Most of the UI is handcoded to look nice on
fullscreen, and some changes will need to be made in order for the game to be pleasing at differnet resolutions.

Another feature that I am considering investing is a server-hosted leaderboard. Right now, players store their scores
in a local leaderboard which shows other players on the machine. However, it would be nice, if the game grows in both 
length and difficulty, to have a server that can display statistics or rankings globally. This networking interface 
might also enable the possibility for multiplayer through timed games, or through custom levels. While these are 
ambitious goals, the rubric by which the game is scored at nationals is pretty strict, and I'll have to conform to 
its requirements in order to maximize my success.

I plan to make another post about these features as time progresses, though it may not be until after the conference. 
As of now, the game not publically avaiable, but is hosted in a private repository on GitHub. As soon as the conference 
is over, which will be roughly the end of June, I'll make the source code public so that any implementations can be 
viewed in full context. 