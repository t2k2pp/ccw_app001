# Stitch Prompt for SlideCraft Desktop Application UI Design (Version 2)

## Overall Design Request

**Generate a series of UI design mockups for a desktop presentation application called "SlideCraft".**

This is a **desktop application for PC**, designed for a widescreen (landscape) monitor. **This is NOT a mobile app or a vertically scrolling website.** The design should be clean, modern, and minimal, with a user-friendly and intuitive interface. Please use a light theme. The primary accent color should be a shade of blue, similar to `#4A89F7`.

Please create **three separate artboards**, one for each of the following screens:
1.  **Dashboard Screen**
2.  **Editor Screen**
3.  **Presentation Screen**

**IMPORTANT: Do NOT stack these artboards vertically to create a single long page. They must be three distinct screens representing different states of the application.**

---

## Artboard 1: Dashboard Screen

**Purpose:** This is the home screen users see when they launch the application. They can create new presentations or open recent ones.

**Layout:**
*   **Artboard Size:** 1600px width, 1200px height.
*   **Header:**
    *   Left: App logo (a simple, abstract icon) and the name "SlideCraft".
    *   Right: A "Settings" icon (cogwheel).
*   **Main Content (Left-aligned):**
    *   **"Create New Slide" Section:**
        *   A prominent heading: "Create New Slide".
        *   Three stacked buttons below the heading:
            1.  "Create from Scratch" (Secondary button style, perhaps white or light gray).
            2.  "Create from Template" (Secondary button style).
            3.  "Create with AI" (Primary button style, filled with the accent blue color, with an icon like a magic wand or stars).
    *   **"Recent Projects" Section:**
        *   A heading: "Recent Projects".
        *   A grid of project thumbnails (e.g., 2 columns, multiple rows).
        *   Each item in the grid should be a card containing:
            *   A preview image of the presentation's first slide.
            *   The project title (e.g., "Project Alpha").
            *   Last updated timestamp (e.g., "Updated 5 minutes ago").

---

## Artboard 2: Editor Screen

**Purpose:** This is the main workspace where users create and edit their slides. It MUST be a three-pane layout to maximize productivity on a widescreen display.

**Layout:**
*   **Artboard Size:** 1920px width, 1080px height.
*   **Overall Structure:** A classic three-pane layout for a desktop application.
    *   **Left Pane (Slide Navigator):** A narrow vertical panel showing a list of slide thumbnails. The currently selected slide should be highlighted. Users can click to navigate and reorder slides here.
    *   **Center Pane (Main Canvas):** The largest area in the middle. This is where the user's slide is displayed and edited. Show a sample slide with a title and a placeholder for an image.
    *   **Right Pane (Properties Inspector):** A vertical panel on the right for editing the properties of selected elements.
*   **Header/Toolbar (at the top of the entire screen):**
    *   Left: An icon to go back to the Dashboard.
    *   Center: The current project's title (e.g., "Project Alpha").
    *   Right: A "Share" icon and a "Present" button (primary button style).
*   **Details of the Right Pane (Properties Inspector):**
    *   It should have three tabs: "Properties", "Design", and "AI".
    *   Under the "Properties" tab (which should be active), show controls for the selected slide (not a text element).
    *   Example controls:
        *   Label: "Background Color" with a color swatch input.
        *   Label: "Template" with a dropdown menu.
        *   Label: "Opacity" with a slider control.
        *   Two buttons at the bottom: "Crop" and "Filter".

---

## Artboard 3: Presentation Screen

**Purpose:** This is the full-screen view for presenting the slides to an audience.

**Layout:**
*   **Artboard Size:** 1920px width, 1080px height.
*   **Content:**
    *   The slide content should be centered and fill most of the screen. Show a beautiful, visually appealing sample slide (e.g., a high-quality image of a beach with a single word "Welcome").
    *   **Controls (overlaying the slide, visible on hover):**
        *   Top Right: An "X" icon to close the presentation view.
        *   Bottom Center: Navigation controls:
            *   A "Prev" button (with a left arrow icon).
            *   A "Next" button (with a right arrow icon).
    *   The background of the artboard outside the slide should be dark to focus attention on the slide itself.

---

## What NOT to do:

*   **Do not create a single, long, scrollable webpage.**
*   **Do not use a mobile-first or mobile-app layout.** This is a desktop-first design.
*   **Do not make the artboards narrow and tall.** They must be wide (landscape).
*   **Avoid ambiguity.** The layout of each screen, especially the 3-pane editor, should be clearly defined as described.
