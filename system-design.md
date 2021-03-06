# ifpress.org

## System Design

### Requirements

 * authors must log in to manage stories
 * users can view without logging in
 * authors have ability to
   * log in to back-end administration system
   * create new projects
   * authorize users to access projects
   * import a ulx or blorb file to a project
   * create/edit templates
     * properties
       * header
       * footer
       * navigation
         * standard
         * custom
       * 1 column
       * 2 columns
       * 3 columns
       * 4 columns
     * identify responsive/mobile/desktop capability
     * layout engine
       * Bootstrap
     * background color
     * foreground color
     * default font
     * mono-spaced font
     * static image(s)
   * edit layout of page for every turn in walk-through
     * create intro page(s)
     * create tutorial page(s)
     * create base page
       * add global content types and designs
     * create pages by filter (inherits base page)
       * default
       * chapter
       * turn range
       * time range
       * custom ('template' content type is returned)
     * add content types to page for selected filter
   * publish story
     * privately
       * author has access
       * selected users have access
     * publicly
     * feedback system enabled/disabled
     * social networking enabled/disabled
 * users have ability to
   * when not logged in
     * browse public stories
     * play public stories
     * comment on public stories
     * chat with others playing same public story
   * when logged in
     * browse public and private stories
     * play public and private stories
     * comment on public and private stories
     * chat with others playing same public or private story
 * JSON-based templates and configuration (not XML)

## Technology

ifpress.org is built on Angular 2, TypeScript, and Boostrap. The hosted service runs on Apache 2 and MongoDB, while the self-hosted version runs on Apache 2 alone. We use MongoDB for user credentials, which isn't required for self-hosting.

## Stories

IF Stories are compiled using Inform 7 (inform7.com). To be compatible with ifpress.org, they will need to include I7 extensions found at the fyrevm-web github repository.

There will be a Core extension that takes care of the basic necessities.

Then there will be additional and optional extensions based on each author's interests.

One of the basic requirements will be for the author to implement ifpress.org settings, so that's going to be a separate extension (you can create FyreVM games without publishing to ifpress.org). Included will be a walk-through (comma separated list of commands to complete game and touch as many parts of the story as possible). This will likely flow through a "ifpress" content type and return JSON.

## Templates

Templates drive the entire display apparatus for ifpress. We start with a base template and go from there. The template system resides within a content area of ifpress.org. If an author is self-hosting, the content area takes up the entire display area. When hosted on ifpress.org, the content area is below a control-panel header.

### Base Template

The base template in ifpress.org will contain communication with the virtual machine, with the ifpress container/cms and back-end services, with local storage, and any other browser management. The base template will be required for all custom templates. It doesn't have any display capabilities on its own.

### Custom Templates

A custom template in ifpress.org inherits the capabilities of the base template, adds its own interactive and display capabilities, and displays a tree of widgets designed by the template creator.

Example:

- Base Template
  - Custom Template for an Interactive Comic Book
    - Header Widget
    - Comic Widget
      - Panel Widget
        - Choice Widget
    - Footer Widget

In this example, we might have a comic book template. The story file tells the template which panel to display relative to any currently displayed panels. The story accepts a choice and updates the panel widget with its changes.

This would allow an author to create an interactive comic book by drawing the art, providing the logic within Inform 7, and using ifpress to publish.

## Pages
## Content Types

### Content Type Mapping (for extension authors)

There is a small technical leap from what is emitted by the virtual machine to being usable by ifpress.org. This is a simple mapping exercise that is automated by a well-known content-type called TYPE. This content type will contain the mappings of any content types defined within a given extension. When a story is loaded, the TYPE content type will contain a JSON structure as shown below:

```
{
  module: 'fyrevm-core.ts',
  mappings: [
    { channel: 'MAIN', contentType: 'main' },
    { channel: 'PRPT', contentType: 'prompt' },
    { channel: 'LOCN', contentType: 'location-name' },
    { channel: 'SCOR', contentType: 'score' },
    { channel: 'TIME', contentType: 'time' },
    { channel: 'DEAD', contentType: 'death' },
    { channel: 'ENDG', contentType: 'end-game' },
    { channel: 'TURN', contentType: 'turn' },
    { channel: 'INFO', contentType: 'story-info' },
    { channel: 'NTFY', contentType: 'score-notify' }
  ]
}
```

### Content Type Module Definitions (for extension authors)

For every extension that defines content types, a TypeScript module is required.

```
module fyreVMCore {

  interface Serializable<T> {
      deserialize(input: Object): T;
  }

  class ContentType implements Serializable<ContentType> {
      channel: string;
      contentType: string;

      deserialize(input) {
          this.channel = input.channel;
          this.contentType = input.contentType;
          return this;
      }
  }

  class ContentTypeDefinition implements Serializable<ContentTypeDefinition> {
      module: string;
      mappings: Array<ContentType>;

      deserialize(input) {
          this.module = input.module;

          for (var mapping in input.mappings) {
            var contentType = new ContentType();
            contentType.channel = mapping.channel;
            contentType.contentType = mapping.contentType;

            this.mappings.add(contentType);
          }

          return this;
      }
  }
}
```
### Base Content Types

Any of these content types can be used in a widget. Note that some content types contain JSON and would require transformation and others are contextually emitted (not available every turn):

 - {{main}} - main text emitted from story progress
 - {{prompt}} - prompt or textual prefix to command line (optional)
 - {{location-name}} - name of current story location
 - {{score}} - current score (optional)
 - {{time}} - current time within story (optional)
 - {{death}} - text emitted upon death within story (optional)
 - {{end-game}} - text emitted when story ends (optional)
 - {{turn}} - number of turns taken in story (optional)
 - {{story-info}} - (JSON) meta data used to build "about" and "credits" information
 - {{score-notify}} - text emitted when score changes (optional)

### ifpress.org Core Content Types (required)

 - {{walk-through}} - (JSON) list of commands to traverse to build content model from uploaded story

### ifpress.org Image Content Type (optional)

 - {{image list}} - (JSON) list of image file names

## Widgets

Widgets are the visual representations of content types found within a given story. This is where each story can display context in visually different ways. Any given content type may have many different related widgets since there are many different ways of displaying the same content.

Some widgets may encapsulate multiple content types too. An example of this might be a standard status line, which contains the location name, the turn, and the score. Here's an example widget in source form:

HTML
```
  <div class="ifp-status-line">
    <span class="ifp-left">{{location-name}}</span>
    <span class="ifp-right">{{score}}</span>
    <span class="ifp-right">{{turn}}</span>
  </div>
```

TypeScript
```
  class Widget {
    element: HTMLElement;
    span: HTMLElement;

    constructor (element: HTMLElement) { 
        this.element = element;
        this.element.class += document.ifpress.ifpStatusLine;
        this.span = document.createElement('span');
        this.span.class = document.ifpress.ifpLeft;
        this.span.innerText = document.ifpress.locationName;
        this.element.appendChild(this.span);
        this.span.class = document.ifpress.ifpRight;
        this.span.innerText = document.ifpress.score;
        this.element.appendChild(this.span);
        this.span.class = document.ifpress.ifpRight;
        this.span.innerText = document.ifpress.turn;
        this.element.appendChild(this.span);
    }
  }
  
  window.onload = () => {
    var el = document.getElementById('content');
    var widget = new Widget(el);
  };
```

## Themes
## Administration
## Publishing
## Security
## Self-Hosting
## Host on ifpress.org


