# @fluid-example/sudoku

This example is a collaborative Sudoku board as a Fluid Container. We used Fluid distributed data structures to store and
synchronize the Sudoku data. We also built a website that loads and renders the Fluid Container.

## Getting Started

To run this follow the steps below:

1. Run `npm install` from the sudoku folder root
2. Run `npm run start` to start both the client and server
3. Navigate to `http://localhost:8080` in a browser tab
4. Copy full URL, including hash id, to a new tab for collaboration

## Acknowledgements

This example uses the [sudokus](https://github.com/Moeriki/node-sudokus) npm package by Dieter Luypaert
(<https://github.com/Moeriki>) and the [@types/sudokus](https://www.npmjs.com/package/@types/sudokus) package by Florian
Keller (<https://github.com/ffflorian>).

## Folder layout

The project has the following layout:

```text
.
├── src
|   ├── app.ts
|   ├── container.ts
|   ├── index.ts
|   ├── fluidSudoku.tsx
|   └── helpers
|       ├── coordinate.ts
|       ├── puzzles.ts
|       ├── styles.css
|       ├── sudokuCell.ts
|       └── react
|           └── sudokuView.tsx
├── public
|   └── index.html
└── tests
    ├── draftjs.test.ts
    ├── index.html
    └── index.ts
```

The `./src/fluidSudoku.tsx` file contains the Sudoku Fluid Data-Object. `./src/container.ts` file contains the Sudoku Fluid Container.

## Available Scripts

### `build`

```bash
npm run build
```

Runs [`tsc`](###-tsc) and [`webpack`](###-webpack) and outputs the results in `./dist`.

### `start`

```bash
npm run start
```

Runs both [`start:client`](###-start:client) and [`start:server`](###-start:server).

### `start:client`

```bash
npm run start:all
```

Uses `webpack-dev-server` to start a local webserver that will host your webpack file.

Once you run `start` you can navigate to `http://localhost:8080` in any browser window to use your fluid example.

> The Tinylicious Fluid server must be running. See [`start:server`](###-start:server) below.

### `start:server`

```bash
npm run start:server
```

Starts an instance of the Tinylicious Fluid server running locally at `http://localhost:3000`.

> Tinylicious only needs to be running once on a machine and can support multiple examples.

### `start:test`

```bash
npm run start:test
```

Uses `webpack-dev-server` to start a local webserver that will host your webpack file.

Once you run `start:test` you can navigate to `http://localhost:8080` in any browser window to test your fluid example.

`start:test` uses a Fluid server with storage to local tab session storage and launches two instances side by side. It does not require Tinylicious.

This is primarily used for testing scenarios.

### `test`

```bash
npm run test
```

Runs end to end test using [Jest](https://jestjs.io/) and [Puppeteer](https://github.com/puppeteer/puppeteer/).

### `test:report`

```bash
npm run test:report
```

Runs `npm run test` with additional properties that will report success/failure to a file in `./nyc/*`. This is used for CI validation.

### `tsc`

Compiles the TypeScript code. Output is written to the `./dist` folder.

### `webpack`

Compiles and webpacks the TypeScript code. Output is written to the `./dist` folder.

## Deep dive

### Data model

For our Sudoku data model, we will use a map-like data structure with string keys. Each key in the map is a coordinate
(row, column) of a cell in the Sudoku puzzle. The top left cell has coordinate `"0,0"`, the cell to its right has
coordinate `"0,1"`, etc.

Each value stored in the map is a `SudokuCell`, a simple class that contains the following properties:

```typescript
value: number // The current value in the cell; 0 denotes an empty cell
isCorrect: boolean = false // True if the value in the cell is correct
readonly fixed: boolean; // True if the value in the cell is supplied as part of the puzzle's "clues"
readonly correctValue: number // Stores the correct value of the cell
readonly coordinate: CoordinateString // The coordinate of the cell, as a comma-separated string, e.g. "2,3"
```

> Objects that are stored in distributed data structures, as `SudokuCell` is, must be safely JSON-serializable. This means
that you cannot use functions or TypeScript class properties with these objects, because those are not JSON-serialized.
>
> One pattern to address this is to define static functions that accept the object as a parameter and manipulate it. See
the `SudokuCell` class in `/src/helpers/sudokuCell.ts` for an example of this pattern.

### Rendering

In order to render the Sudoku data, we use a React component called `SudokuView` This component is defined in
`src/react/sudokuView.tsx` and accepts the map of Sudoku cell data as a prop. It then renders the Sudoku and
accompanying UI.

The `SudokuView` React component is also responsible for handling UI interaction from the user; we'll examine that in
more detail later.

### The Fluid Object

The React component described above does not itself represent a Fluid FluidObject. Rather, the Fluid Object is defined
in `src/fluidSudoku.tsx`.

```typescript
export class FluidSudoku extends DataObject implements IFluidHTMLView {}
```

This class extends the [DataObject][] abstract base class. Our FluidObject is visual, so we need to implement the
[IFluidHTMLView][] interface. In our case, we want to handle rendering ourselves rather than delegate it to another object,
so we implement [IFluidHTMLView][].

#### Implementing interfaces

##### IFluidHTMLView

[IFluidHTMLView][] requires us to implement the `render()` method, which is straightforward since we're using the
`SudokuView` React component to do the heavy lifting.

```typescript
public render(element?: HTMLElement): void {
    if (element) {
        this.domElement = element;
    }
    if (this.domElement) {
        let view: JSX.Element;
        if (this.puzzle) {
            view = (
                <SudokuView
                    puzzle={this.puzzle}
                    clientPresence={this.clientPresence}
                    clientId={this.runtime.clientId ?? "not connected"}
                    setPresence={this.presenceSetter}
                />
            );
        } else {
            view = <div />;
        }
        ReactDOM.render(view, this.domElement);
    }
}
```

As you can see, the render method uses React to render the `SudokuView` React component.  Notice that we pass the
puzzle data, a `SharedMap` distributed data structure that we will discuss more below, to the SudokuView React
component as props.

#### Creating Fluid distributed data structures

How does the `puzzle` property get populated? How are distributed data structures created and used?

To answer that question, look at the `initializingFirstTime` method in the `FluidSudoku` class:

```typescript
private sudokuMapKey = "sudoku-map";
private puzzle: ISharedMap;

protected async initializingFirstTime() {
    // Create a new map for our Sudoku data
    const map = SharedMap.create(this.runtime);

    // Populate it with some puzzle data
    loadPuzzle(0, map);

    // Store the new map under the sudokuMapKey key in the root SharedDirectory
    this.root.set(this.sudokuMapKey, map.handle);
}
```

This method is called once when a FluidObject is initially created. We create a new [SharedMap][] using `.create`,
registering it with the runtime. We have access to the Fluid runtime from `this.runtime` because we have subclassed
[DataObject][].

Once the SharedMap is created, we populate it with puzzle data. Finally, we store the SharedMap we just created in the
`root` [SharedDirectory][]. The `root` [SharedDirectory][] is provided by [DataObject][], and is a convenient place
to store all Fluid data used by your FluidObject.

Notice that we provide a string key, `this.sudokuMapKey`, when we store the `SharedMap`. This is how we will retrieve
the data structure from the root SharedDirectory later.

`initializingFirstTime` is only called the _first time_ the FluidObject is created. This is exactly what we want
in order to create the distributed data structures. We don't want to create new SharedMaps every time a client loads the
FluidObject! However, we do need to _load_ the distributed data structures each time the FluidObject is loaded.

Distributed data structures are initialized asynchronously, so we need to retrieve them from within an asynchronous
method. We do that by overloading the `hasInitialized` method, then store a local reference to the object
(`this.puzzle`) so we can easily use it in synchronous code.

```typescript
protected async hasInitialized() {
    this.puzzle = await this.root.get<IFluidHandle>(this.sudokuMapKey).get<ISharedMap>();
}
```

The `hasInitialized` method is called once after the FluidObject has completed initialization, be it the first
time or subsequent times.

##### A note about FluidObject handles

You probably noticed some confusing code above. What are handles? Why do we store the SharedMap's _handle_ in the `root`
SharedDirectory instead of the SharedMap itself? The underlying reasons are beyond the scope of this example, but the
important thing to remember is this:

**When you store a distributed data structure within another distributed data structure, you store the _handle_ to the
DDS, not the DDS itself. Similarly, when loading a DDS that is stored within another DDS, you must first get the DDS
handle, then get the full DDS from the handle.**

```typescript
await this.root.get<IFluidHandle>(this.sudokuMapKey).get<ISharedMap>();
```

#### Handling events from distributed data structures

Distributed data structures can be changed by both local code and remote clients. In the `hasInitialized`
method, we also connect a method to be called each time the Sudoku data - the [SharedMap][] - is changed. In our case we
simply call `render` again. This ensures that our UI updates whenever a remote client changes the Sudoku data.

```typescript
this.puzzle.on("valueChanged", (changed, local, op) => {
  this.render();
});
```

#### Updating distributed data structures

In the previous step we showed how to use event listeners with distributed data structures to respond to remote data
changes. But how do we update the data based on _user_ input? To do that, we need to listen to some DOM events as users
enter data in the Sudoku cells. Since the `SudokuView` class handles the rendering, that's where the DOM events will be
handled.

Let's look at the `numericInput` function, which is called when the user keys in a number.

::: note

The `numericInput` function can be found in the `SimpleTable` React component within `src/react/sudokuView.tsx`.
`SimpleTable` is a helper React component that is not exported; you can consider it part of the `SudokuView` React
component.

:::

```typescript{2-6,9}
const numericInput = (keyString: string, coord: string) => {
  let valueToSet = Number(keyString);
  valueToSet = Number.isNaN(valueToSet) ? 0 : valueToSet;
  if (valueToSet >= 10 || valueToSet < 0) {
    return;
  }

  if (coord !== undefined) {
    const cellInputElement = getCellInputElement(coord);
    cellInputElement.value = keyString;

    const toSet = props.puzzle.get<SudokuCell>(coord);
    if (toSet.fixed) {
      return;
    }
    toSet.value = valueToSet;
    toSet.isCorrect = valueToSet === toSet.correctValue;
    props.puzzle.set(coord, toSet);
  }
};
```

Lines 2-6 ensure we only accept single-digit numeric values. In line 9, we retrieve the coordinate of the cell from a DOM
attribute that we added during render. Once we have the coordinate, which is a key in the `SharedMap` storing our Sudoku
data, we retrieve the cell data by calling `.get<SudokuCell>(coord)`. We then update the cell's value and set whether it
is correct. Finally, we call `.set(key, toSet)` to update the data in the `SharedMap`.

This pattern of first retrieving an object from a `SharedMap`, updating it, then setting it again, is an idiomatic Fluid
pattern. Without calling `.set()`, other clients will not be notified of the updates to the values within the map. By
setting the value, we ensure that Fluid notifies all other clients of the change.

Once the value is set, the `valueChanged` event will be raised on the SharedMap, and as you'll recall from the previous
section, we listen to that event and render again every time the values change. Both local and remote clients will
render based on this event, because all clients are running the same code.

**This is an important design principle:** FluidObjects should have the same logic for handling local and remote changes.
In other words, it is very rare that there is a need for the handling to differ, and we recommend a unidirectional data
flow.

## Lab: Adding "presence" to the Fluid Sudoku FluidObjects

The Sudoku FluidObject is collaborative; multiple clients can update the cells in real time. However, there's no
indication of where other clients are - which cells they're in. In this lab we'll add basic 'presence' to our Sudoku
FluidObject, so we can see where other clients are.

To do this, we'll create a new `SharedMap` to store the presence information. Like the map we're using for Sudoku data,
it will be a map of cell coordinates to client names. As clients select cells, the presence map will be updated with the
current client in the cell.

Note that using a SharedMap for presence means that the history of each user's movement - their presence - will be
persisted in the Fluid op stream. In the Sudoku scenario, maintaining a history of a client's movement isn't
particularly interesting, and Fluid provides an alternative mechanism, _signals_, to address cases where persisting ops
isn't necessary. That said, this serves as a useful example of how to use Fluid to solve complex problems with very
little code.

### Create a SharedMap to contain presence data

First, you need to create a `SharedMap` for your presence data.

1. Open `src/fluidSudoku.tsx`.
1. Inside the `FluidSudoku` class, declare two new private variables like so:

   ```ts
   private readonly presenceMapKey = "clientPresence";
   private clientPresence: ISharedMap | undefined;
   ```

1. Inside the `initializingFirstTime` method, add the following code to the bottom of the method to create and
   register a second `SharedMap`:

   ```ts
   // Create a SharedMap to store presence data
   const clientPresence = SharedMap.create(this.runtime);
   this.root.set(this.presenceMapKey, clientPresence.handle);
   ```

   Notice that the Fluid runtime is exposed via the `this.runtime` property provided by [DataObject][].

1. Inside the `hasInitialized` method, add the following code to the bottom of the method to retrieve the
   presence map when the FluidObject initializes:

   ```ts
   this.clientPresence = await this.root
     .get<IComponentHandle>(this.presenceMapKey)
     .get<ISharedMap>();
   ```

You now have a `SharedMap` to store presence data. When the FluidObject is first created, `initializingFirstTime`
will be called and the presence map will be created. When the FluidObject is loaded, `hasInitialized` will be
called, which retrieves the `SharedMap` instance.

### Rendering presence

Now that you have a presence map, you need to render some indication that a remote user is in a cell. We're going to
take a shortcut here because our SudokuView React component can already display presence information when provided two
optional props:

```ts
clientPresence?: ISharedMap;
setPresence?(cellCoord: CoordinateString, reset: boolean): void;
```

We aren't providing those props, so the presence display capabilities within the React component aren't enabled. After
you've completed this tutorial, you should consider reviewing the implementation of the presence rendering within
SudokuView in detail. For now, however, we'll skip that and focus on implementing the two necessary props - a SharedMap
for storing the presence data, and a function to update the map with presence data.

### Setting presence data

1. Open `src/fluidSudoku.tsx`.
1. Add the following function at the bottom of the `FluidSudoku` class:

   ```ts
   /**
    * A function that can be used to update presence data.
    *
    * @param cellCoordinate - The coordinate of the cell to set.
    * @param reset - If true, presence for the cell will be cleared.
    */
   private readonly presenceSetter = (cellCoordinate: string, reset: boolean): void => {
       if (this.clientPresence) {
           if (reset) {
               // Retrieve the current clientId in the cell, if there is one
               const prev = this.clientPresence.get<string>(cellCoordinate);
               const isCurrentClient = this.runtime.clientId === prev;
               if (!isCurrentClient) {
                   return;
               }
               this.clientPresence.delete(cellCoordinate);
           } else {
               this.clientPresence.set(cellCoordinate, this.runtime.clientId);
           }
       }
   };
   ```

   You can pass this function in to the `SudokuView` React component as a prop. The React component will call
   `presenceSetter` when users enter and leave cells, which will update the presence `SharedMap`.

1. Replace the `createJSXElement` method with the following code:

   ```ts
   public createJSXElement(): JSX.Element {
       if (this.puzzle) {
           return (
               <SudokuView
                   puzzle={this.puzzle}
                   clientPresence={this.clientPresence}
                   clientId={this.runtime.clientId}
                   setPresence={this.presenceSetter}
               />
           );
       } else {
           return <div />;
       }
   }
   ```

   Notice that we're now passing the `clientPresence` SharedMap and the `setPresence` function as props.

### Listening to distributed data structure events

1. Still in `src/fluidSudoku.tsx`, add the following code to the bottom of the `hasInitialized` method to call
   render whenever a remote change is made to the presence map:

   ```ts
   this.clientPresence.on("valueChanged", (changed, local, op) => {
     this.render();
   });
   ```

### Testing the changes

Now run `npm start` again and notice that your selected cell is now highlighted on the other side.

## Next steps

Now that you have some experience with Fluid, are there other features you could add to the Sudoku FluidObject? Perhaps
you could extend it to display a client name in the cell to show client-specific presence. Or you could use the
[undo-redo][] package to add undo/redo support!
