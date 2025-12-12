---
title: "Hacking Forager — Part 2: Creating a ImGui Overlay"
date: 2025-12-11
categories: [gamehacking, low-level]
tags: [imgui,hooking,directx]     # TAG names should always be lowercase
---

### Introduction

In [Part 1](https://archyinuse.github.io/posts/Hacking-Forager-1/), I injected a DLL into Forager and prepared the groundwork for modifying the game internally (namely, ensuring code execution within the context of Forager).  
In this second part, I'll be focusing on building an in-game interface for the cheats: an ImGui overlay, rendered through a DirectX11 hook.

This overlay is essential for a good user experience (the user being me, of course), aswell as create input fields that I wouldn't have control over without a proper GUI.  
To achieve this, we'll be taking a look at:
- Hooking into Forager’s DirectX rendering pipeline
- Initializing ImGui inside the game
- Intercepting Win32 input events
- Drawing a functional UI on top of the game

This post explains parts of that process using excerpts from the code found [on github](https://github.com/ArchyInUse/ForagerCheatMenu).
Particularly [`dllmain.cpp`](https://github.com/ArchyInUse/ForagerCheatMenu/blob/master/CheatMenu/dllmain.cpp), where hooking DX11 and the main ImGui input loop occurs, aswell as parts of [`controller.cpp`](https://github.com/ArchyInUse/ForagerCheatMenu/blob/master/CheatMenu/Controller.cpp) where the GUI is drawn.

> A quick note: While I do my best to research for these posts; I am far from an expert in DX11, graphics APIs & pipelines in general, if I get anything wrong in this post, please let me know by messaging me on [GitHub](https://github.com/ArchyInUse).

---

### 1. Intercepting Input Through a WndProc Hook
Forager runs on a standard Win32 window, which means we can intercept input by replacing its window procedure. Windows delivers all events as messages to a `WndProc` callback function with the following signature:
`LRESULT CALLBACK WndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam);`
The interesting argument here is `uMsg` as it signifies the message. (A full list of messages can be found [here](https://learn.microsoft.com/en-us/windows/win32/winmsg/about-messages-and-message-queues#system-defined-messages))

We are replacing this window procedure on this line:
`g_OldWndProc = (WNDPROC)SetWindowLongW(gameWnd, GWLP_WNDPROC, (LONG_PTR)HookedWndProc);`

Where `HookedWndProc` is a function that handles the events with custom behavior for our overlay:
```cpp
LRESULT CALLBACK HookedWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
	if (ImGui_ImplWin32_WndProcHandler(hWnd, uMsg, wParam, lParam))
		return 0;

	// block game from getting mouse inputs when menu is visible
	if (Controller::Get().MenuVisible()) {
		ImGuiIO& io = ImGui::GetIO();
		const bool mouse_msg =
			uMsg == WM_LBUTTONDOWN || uMsg == WM_LBUTTONUP ||
			uMsg == WM_RBUTTONDOWN || uMsg == WM_RBUTTONUP ||
			uMsg == WM_MBUTTONDOWN || uMsg == WM_MBUTTONUP ||
			uMsg == WM_MOUSEMOVE || uMsg == WM_MOUSEWHEEL || uMsg == WM_MOUSEHWHEEL;
		const bool key_msg = (uMsg == WM_KEYDOWN || uMsg == WM_KEYUP || uMsg == WM_CHAR);

		if ((mouse_msg && io.WantCaptureMouse) || (key_msg && io.WantCaptureKeyboard))
			return 0; 
	}
  
  // if the menu isn't visible, call the old window proc and let the game handle input normally.
	return CallWindowProc(g_OldWndProc, hWnd, uMsg, wParam, lParam);
}
```

After hooking, all events for the game window go through the custom window procedure.

Forwarding messages to ImGui:\
`if (ImGui_ImplWin32_WndProcHandler(hWnd, uMsg, wParam, lParam)) return 0;`\
`ImGuiImplWin32_WndProcHandler({...});` is a backend function for ImGui. In essence, it looks at `uMsg` and decides whether the event is relevant for the overlay. This is used for things such as pressing a text field where the expected behavior is for the user to be able to write a string. In cases like this, ImGui eats up the event and doesn't pass it onto Forager.

The function returns `true` if ImGui fully handled the message and false otherwise. 

The previous function also updates `ImGui::GetIO()` which we use later on as another layer of event filtering.
`if ((mouse_msg && io.WantCaptureMouse) || (key_msg && io.WantCaptureKeyboard)) return 0;`

This might seem a little convoluted at first, but this is where it's important to understand how the backend works. The ImGui handler eats up events such as typed characters, mouse wheel events and some button presses depending on context, ImGui only processes absolutely necessary events to run it's UI, but not all *relevant* events for our case.
Certain key down events, aswell as moving our mouse over the overlay would result in double input, one for ImGui and one for Forager which would make game interaction difficult.

To explicitely make the overlay take full ownership based on UI state, we add the second layer, for example, if the menu is open and we're hovering over a slider, ImGui will set `io.WantCaptureMouse = true;` but it *won't* consume every mouse movement message, which would create double inputs. 

---

### 2. Creating a Render Target for the Overlay
To draw ImGui’s output, we need a render target view (RTV).  

A render target is a texture that Direct3D is allowed to draw into. Typically, the main render target is the back buffer, which contains the image that is going to be presented to the screen next.

We want to draw the overlay on top of Forager's final frame, so we'll be reusing Forager's back buffer as our drawing surface, creating `m_mainRTV` for drawing.

```cpp
m_swap->GetBuffer(0, __uuidof(ID3D11Texture2D), (void**)&pBackBuffer);
m_device->CreateRenderTargetView(pBackBuffer, nullptr, &m_mainRTV);
```

While this post won't get too much into DX11 ImGui initialization (more on that in the next section), if you want to check out the code for the initialization, which is the part where DX11 is hooked via [MinHook](https://github.com/TsudaKageyu/minhook), a swap chain and DX11 device and are created, you can check it out [here](https://github.com/ArchyInUse/ForagerCheatMenu/blob/10a24b9743651088ff514b807a4e49bbcf0262d2/CheatMenu/dllmain.cpp#L102). 

---

### 3. Initializing ImGui Inside Forager and Hooking Render Loops
Unfortunately, I won’t be diving into the full details of how ImGui is initialized inside Forager (a DX11 executable). This topic becomes very deep very quickly, and these posts aim to teach hacking concepts—not the full DirectX 11 rendering pipeline.

While we will be hooking functions later, these posts are not intended to be DX11 primers. That said, I still want to shout out several fantastic resources that helped me understand the DX11 side of things: 

- [Fred Emmot - In-Game Overlays: How They Work](https://fredemmott.com/blog/2022/05/31/in-game-overlays.html) Provides a high level look at how in-game overlays work.
- [Kyle Halladay - Hooking and Hijacking DX11 Functions in Skyrim](https://kylehalladay.com/blog/2021/07/14/Dll-Search-Order-Hijacking-For-PostProcess-Injection.html) - Provides very useful code samples and examples of hooking DX11 functions to override graphical behavior of Skyrim.
- [Andrea Venuta - Sekiro Practice Tool](https://veeenu.github.io/blog/sekiro-practice-tool-architecture/) - This post was the backbone of most of the DX11 code and provides a very good starting point for hooking DX11.
- [Sh0ckFR - Github Project on Hooking DX10/11](https://github.com/Sh0ckFR/Universal-ImGui-D3D11-Hook) - This project can be used to sample code for hooking. While I didn't use this at the time ; I found it later on and wanted to add this to the relevant sources.

---

### 4. High Level Overview - Hooking DX11
Regardless of the last section, I do want to provide a high level overview of what's being done so we have a baseline understanding without diving into DX11 internals.

1. Hook the game's DX11 `Present` function
This gives us a callback every frame, just before the rendered image (Forager's frame) is displayed. We'll need access to that fully rendered frame so we can draw the overlay on top of it.
2. Extract the DX11 device, context, and swap chain
These are DirectX's core rendering objects, we'll be passing them to ImGui so that it can draw the overlay for us.
3. Create a render target view
This wraps the game's back buffer (the buffer of the next displayed frame) so ImGui has a surface to draw onto.
4. Initialize ImGui with the Win32 + DX11 backends
Once initialized, ImGui becomes "render-ready" inside the game's frame loop.
5. Render ImGui each frame from inside our hooked `Present` function
By doing this after the game finishes render, the overlay is drawn on top of the frame.

This is a *very* high level overview of the DX11 side of things. From here on, everything shifts back to C++ ImGui rendering, which is considerably simpler than DX11 rendering, as ImGui provides utility functions for components and, at this point, handles the rendering for us.

### 5. Creating the Cheat Menu UI  
The menu displays real-time stats pulled from pointer chains (which we'll discover and use in the next post):
```cpp
ImGui::Text("Health: %lf", *data->health); 
ImGui::Text("Money: %lf", *data->money); 
ImGui::Text("Stamina: %lf", *data->stamina); 
ImGui::Text("Damage Multiplier: %lf", *data->damage);
```

> Note: most of Forager's in game data are of type `double`. While I can't say for sure, I suspect the reason for this is for animations via interpolation. Value interpolation is used widely in Forager, possibly linear interpolation (also called `lerp` for short), which would require decimal precision for the effect, especially at the early stages where the game doesn't provide you with large amounts of money, stamina and xp to work with.

![Debug Values](/assets/image/2025-12-11/debugVals.png)

#### Example: Toggles
In ImGui, the `ImGui::Checkbox` widget is defined as:\
`bool ImGui::Checkbox(const char* label, bool* v)`\
Where the label is a display name but in our case is a unique identifier (denoted by `##`), and `v` is a pointer to the boolean state.

ImGui uses an immediate mode render model, which means it does not store state internally. Instead, it asks you to provide a pointer to the variable that owns the state and updates that value directly, we'll see this pattern in other widgets aswell.

The boolean ImGui returns to us is if the user clicked the widget *this frame*, this part of ImGui's behavior is not applicable to this example but is relevant in the project when we want to turn something on or off with more complex behavior, we'll see more of this behavior when we start patching runtime behavior within the executable. 

```cpp
ImGui::Checkbox("##lockStamina", &m_settings->lockStamina); // create widget
if (m_settings->lockStamina) 
    *this->m_data->stamina = maxValues::MAX_STAMINA; // perform action if the widget is selected
```

#### Example: Buttons
The button widget, similarly to the checkbox returns `true` when the widget is clicked. For buttons we'll only need to check when it's clicked only on the frame it's clicked.\
`if (ImGui::Button("Double Gold"))     *this->m_data->money *= 2;`

#### Example: Editable
`ImGui::Input{type}` returns `true` when the user modifies the value in this frame. For those wondering why I make a copy of the money data each frame, this is a safety measure for dynamic values for games. If ImGui changes the value while Forager is also trying to change the value, it could cause [resource contention](https://en.wikipedia.org/wiki/Resource_contention) which, in turn, could cause a crash or other weird behavior for the game.
```cpp
double moneyCopy = *this->m_data->money;  
if (ImGui::InputDouble("##money", &moneyCopy))     
    *this->m_data->money = moneyCopy;
```

All together, the overlay provides us with a good interface to start implementing the cheats.
![full screenshot of the overlay](/assets/image/2025-12-11/fullSS.png)

> Note: the development cycle is not the same as shown in the posts; After setting up ImGui, I used it for extensive testing of finding the correct values, making each cheat with a UI component together, not separately.

---

### Conclusion

This overlay serves as the primary user interface for all future debugging, memory editing, and gameplay manipulation tools in the project.

In part 3, we will expand on how game memory works on a deeper level, finding static pointers and resolving static pointer chains, which are some of the most exciting (more on that in the next section)g parts of game hacking!

Stay tuned and thanks for reading!
