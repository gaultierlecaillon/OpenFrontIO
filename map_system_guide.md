# Understanding the Map System in OpenFrontIO

This guide explains how the map system works in OpenFrontIO and provides instructions on how to create your own custom maps for the game.

## Table of Contents

1. [Overview](#overview)
2. [Map Structure](#map-structure)
3. [Creating a Custom Map](#creating-a-custom-map)
4. [Map Generation Process](#map-generation-process)
5. [Advanced Customization](#advanced-customization)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

## Overview

The map system in OpenFrontIO uses a combination of image files and JSON configuration to create playable maps. Maps are processed by a map generator tool that converts the raw image and configuration into optimized binary files used by the game.

Key components:

- **Image file**: Defines the land and water areas of the map
- **Info JSON**: Defines the nations, their starting positions, and other map metadata
- **Map Generator**: Processes these files to create the game-ready map files

## Map Structure

### Source Files

Each map consists of two primary source files:

1. **image.png**: An image file where:

   - Transparent pixels or specific blue color (RGB value with blue=106) represent water
   - Other pixels represent land
   - The blue channel (140-200 range) is used to determine terrain elevation/magnitude

2. **info.json**: A JSON configuration file that defines:
   - Map name
   - Nations (countries/regions) on the map
   - Each nation's starting coordinates, name, strength, and flag

Example `info.json`:

```json
{
  "name": "Iceland",
  "nations": [
    {
      "coordinates": [455, 1115],
      "name": "Southern Peninsula",
      "strength": 2,
      "flag": "is"
    },
    {
      "coordinates": [550, 1050],
      "name": "Capital Region",
      "strength": 2,
      "flag": "is"
    }
    // More nations...
  ]
}
```

### Generated Files

The map generator creates several output files:

1. **map.bin**: Binary data representing the full-resolution map
2. **mini_map.bin**: Binary data for a half-resolution version of the map
3. **thumbnail.webp**: A visual preview of the map
4. **manifest.json**: Enhanced version of info.json with additional metadata

## Creating a Custom Map

Follow these steps to create your own custom map:

### Step 1: Set Up Your Environment

1. Make sure you have Go installed (https://go.dev/doc/install)
2. Clone the OpenFrontIO repository
3. Navigate to the `map-generator` directory

### Step 2: Create Map Directory

1. Create a new folder in `map-generator/assets/maps/<your_map_name>`

### Step 3: Create the Image File

1. Create an `image.png` file for your map:
   - You can start with a real-world map and modify it
   - For performance reasons, aim for around 2 million pixels (e.g., 1414×1414)
   - Do not exceed 4 million pixels
   - Use transparency or specific blue color (blue=106) for water areas
   - Use the blue channel (140-200) to indicate terrain elevation:
     - 140-150: Plains (lower elevation)
     - 150-170: Highlands (medium elevation)
     - 170-200: Mountains (higher elevation)

### Step 4: Create the Info JSON

1. Create an `info.json` file with the following structure:

```json
{
  "name": "Your Map Name",
  "nations": [
    {
      "coordinates": [x, y],
      "name": "Nation Name",
      "strength": 1-5,
      "flag": "country_code"
    },
    // Add more nations...
  ]
}
```

Notes:

- `coordinates`: [x, y] position on the map image where the nation starts
- `strength`: Initial military strength (typically 1-5)
- `flag`: Country code for the flag (use ISO 3166 country codes, e.g., "us", "uk", "fr")
  - You can find the available flags in the `resources/flags/` directory

### Step 5: Register Your Map

1. Edit `map-generator/main.go` to add your map to the `maps` array:

```go
var maps = []struct {
    Name   string
    IsTest bool
}{
    // Existing maps...
    {Name: "your_map_name"},
}
```

### Step 6: Generate the Map

1. Run the map generator:

```
cd map-generator
go run .
```

2. The generator will process your map and create the output files in `resources/maps/<your_map_name>/`

## Map Generation Process

Understanding how maps are processed can help you create better maps:

1. **Image Processing**:

   - The PNG image is decoded and processed pixel by pixel
   - Land and water areas are identified
   - Small islands (< 30 pixels) are automatically removed
   - Small bodies of water (< 200 pixels) are filled in
   - Shorelines are identified (land adjacent to water and vice versa)
   - Water tiles are assigned a "distance from land" value

2. **Binary Encoding**:

   - Each tile is encoded into a single byte:
     - Bit 7: Land (1) or Water (0)
     - Bit 6: Shoreline (1) or not (0)
     - Bit 5: Ocean (1) or not (0) (largest water body)
     - Bits 0-4: Magnitude value (elevation for land, distance from shore for water)

3. **Output Generation**:
   - Full-resolution map binary
   - Half-resolution mini-map binary
   - Thumbnail image
   - Updated manifest with map dimensions and statistics

## Advanced Customization

### Terrain Types

The map system supports different terrain types based on the blue channel value:

- **Water**:

  - Shoreline water: Special rendering for water tiles adjacent to land
  - Ocean water: Deeper blue based on distance from shore

- **Land**:
  - Plains (magnitude < 10): Lighter green/brown
  - Highlands (magnitude 10-20): Medium elevation terrain
  - Mountains (magnitude > 20): Higher elevation, rendered with lighter colors

### RGB Color Values Explained

The map generator uses specific RGB values to determine terrain types:

- **Water Areas**:

  - Can be transparent pixels (alpha = 0)
  - Or pixels with blue channel value of 106 (RGB: any, any, 106)
  - Example: RGB(0, 0, 106) or RGBA(0, 0, 106, 255)

- **Land Elevation**:

  - The blue channel value between 140-200 determines the terrain elevation
  - Plains: Blue channel 140-150 (e.g., RGB(any, any, 140-150))
  - Highlands: Blue channel 150-170 (e.g., RGB(any, any, 150-170))
  - Mountains: Blue channel 170-200 (e.g., RGB(any, any, 170-200))
  - Example for plains: RGB(190, 220, 145)
  - Example for highlands: RGB(200, 183, 160)
  - Example for mountains: RGB(230, 230, 185)

  water: 00006a

Note that while the red and green channels can be any value, the map generator primarily uses the blue channel for terrain determination. The actual rendering colors in the game will be different from these input colors.

### Custom Flags

If you want to use custom flags:

1. Add your SVG flag files to the `resources/flags/` directory
2. Reference them in your `info.json` using the filename without the `.svg` extension

## Best Practices

1. **Map Size**:

   - Aim for around 2 million pixels for optimal performance
   - Larger maps (up to 4 million pixels) will work but may affect performance

2. **Nation Placement**:

   - Place nations on land, not water
   - Distribute nations evenly across the map
   - Consider geographical barriers (mountains, water) for strategic gameplay

3. **Testing**:

   - Test your map with different numbers of players
   - Ensure all nations have reasonable access to expansion

4. **Image Preparation**:
   - Use image editing software like GIMP or Photoshop
   - Ensure clean edges between land and water
   - Use the blue channel intentionally to create terrain variation

## Multiplayer Testing

When you want to test your custom maps with friends on other networks, you'll need to make your development server accessible from the internet. Here are some options:

### Option 1: Port Forwarding

1. Configure your router to forward port 9000 (or whichever port your server uses) to your computer's local IP address
2. Share your public IP address with your friends
3. They can connect using http://your-public-ip:9000

### Option 2: Use a Tunneling Service

Tunneling services provide a simple way to expose your local development server to the internet without configuring your router.

1. **Install ngrok**:

   ```
   npm install -g ngrok
   ```

2. **Start your development server** (if it's not already running):

   ```
   npm run dev
   ```

3. **Create a tunnel to your local server**:

   ```
   ngrok http 9000 --host-header=localhost
   ```

4. **Share the generated URL with your friends**:

   - Ngrok will provide a URL like `https://a1b2c3d4.ngrok.io`
   - This URL will forward to your local development server

5. **Keep the ngrok terminal window open** while testing with friends

Other tunneling options include:

- Cloudflare Tunnel
- Localtunnel
- Pagekite

### Option 3: Deploy to a Public Server

For a more permanent solution, you can build and deploy the game to a public server:

1. Build the project for production:

   ```
   npm run build
   ```

2. Deploy the built files to a web hosting service of your choice

## Troubleshooting

### Common Issues

1. **Map Not Appearing in Game**:

   - Ensure your map is properly registered in `main.go`
   - Check that all required files are generated in the `resources/maps/<your_map_name>/` directory
   - Verify that your map is registered in the client-side code:
     - Check that your map is added to the `MapDescription` object in `src/client/components/Maps.ts`
     - Ensure your map is added to the `GameMapType` enum in the game code
     - Make sure your map is included in the `numPlayersConfig` in `src/core/configuration/DefaultConfig.ts`
   - Restart the game server after generating the map to ensure it loads the new map files

2. **Nations Not Spawning Correctly**:

   - Verify coordinates in `info.json` point to land tiles (not water)
   - Check that the coordinates are within the bounds of your image
   - Ensure each nation has a valid flag code that exists in the `resources/flags/` directory

3. **Map Generation Errors**:

   - Ensure your image is in PNG format
   - Check that your `info.json` is valid JSON (no trailing commas, properly formatted)
   - Make sure your image size is reasonable (under 4 million pixels)
   - If you're getting errors during map generation, try running the generator with more verbose output:
     ```
     cd map-generator && go run . -v
     ```
   - Check that your image uses the correct RGB values for water and terrain elevation

4. **Only "plains" Map Being Generated**:

   - This is often caused by concurrency issues in the map generator
   - Try modifying the `loadTerrainMaps()` function in `main.go` to process maps sequentially instead of concurrently
   - Make sure your map directory and files have the correct permissions

5. **Performance Issues**:
   - Reduce the size of your map image
   - Simplify coastlines and remove small details
   - Optimize the number of nations based on the map size

### Complete Step-by-Step Guide

Here's a complete walkthrough for creating a custom map:

1. **Create the map directory**:

   ```
   mkdir -p map-generator/assets/maps/your_map_name
   ```

2. **Create or prepare your image**:

   - Create an `image.png` file with appropriate dimensions (aim for around 2 million pixels)
   - Use transparent pixels or RGB(any, any, 106) for water areas
   - Use the blue channel (140-200) to indicate terrain elevation

3. **Create the info.json file**:

   ```json
   {
     "name": "Your Map Name",
     "nations": [
       {
         "coordinates": [x, y],
         "name": "Nation Name",
         "strength": 2,
         "flag": "country_code"
       },
       // Add more nations...
     ]
   }
   ```

4. **Register your map in main.go**:

   - Edit `map-generator/main.go` to add your map to the `maps` array:

   ```go
   var maps = []struct {
       Name   string
       IsTest bool
   }{
       // Existing maps...
       {Name: "your_map_name"},
   }
   ```

5. **Register your map in client code**:

   - Add your map to the `GameMapType` enum (if not already there)
   - Add your map to the `MapDescription` object in `src/client/components/Maps.ts`:

   ```typescript
   export const MapDescription: Record<keyof typeof GameMapType, string> = {
     // Existing maps...
     YourMapName: "Your Map Display Name",
   };
   ```

   - Add your map to the `numPlayersConfig` in `src/core/configuration/DefaultConfig.ts`:

   ```typescript
   const numPlayersConfig = {
     // Existing maps...
     [GameMapType.YourMapName]: [50, 30, 20], // [large, medium, small] player counts
   } as const satisfies Record<GameMapType, [number, number, number]>;
   ```

6. **Generate the map**:

   ```
   cd map-generator
   go run .
   ```

7. **Verify the output**:

   - Check that the following files were created in `resources/maps/your_map_name/`:
     - `map.bin`
     - `mini_map.bin`
     - `thumbnail.webp`
     - `manifest.json`

8. **Restart the game server** to load the new map

### Getting Help

If you encounter issues not covered in this guide, you can:

- Check the OpenFrontIO documentation
- Look at existing maps for examples
- Examine the map generator code to understand how maps are processed
- Ask for help in the community forums or Discord

---

Happy map-making! With this guide, you should be able to create custom maps for OpenFrontIO and understand how the map system works under the hood.
