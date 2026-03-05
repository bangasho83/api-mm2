# Image-to-Prompt API

## Overview

The Image-to-Prompt API analyzes an image and generates an ultra-detailed, professional prompt that can be used to recreate the image using AI image generation models. It uses OpenAI's GPT-4o Vision model to extract comprehensive visual details and structured metadata.

## Endpoint

```
POST /api/image-to-prompt
```

## Request

### Headers
```
Content-Type: application/json
```

### Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `imageUrl` or `image_url` | string | Yes | URL of the image to analyze |
| `userId` | string | No | User ID for tracking purposes |
| `organizationId` | string | No | Organization ID for tracking purposes |

### Example Request

```json
{
  "imageUrl": "https://example.com/product-image.jpg",
  "userId": "user123",
  "organizationId": "org456"
}
```

## Response

### Success Response (200)

```json
{
  "success": true,
  "prompt": "A photorealistic product shot featuring a sleek, modern smartphone positioned at a 45-degree angle on a minimalist white marble surface. The device displays a vibrant gradient screen with deep blues transitioning to purples. Soft, diffused studio lighting creates subtle highlights along the phone's polished aluminum edges and casts a gentle shadow beneath. The composition follows the rule of thirds with the phone positioned slightly off-center. Background features a clean, out-of-focus gradient from pure white to light gray. Camera settings simulate a professional DSLR with shallow depth of field (f/2.8), creating beautiful bokeh in the background while maintaining sharp focus on the device's screen and front face...",
  "variables": {
    "subject": "Modern smartphone",
    "style": "photorealistic",
    "lighting": "soft studio lighting with diffused highlights",
    "color_palette": ["white", "silver", "blue", "purple", "gray"],
    "mood": "professional, clean, modern",
    "composition": "rule of thirds, slightly off-center",
    "camera_angle": "45-degree angle, eye-level perspective",
    "background": "minimalist white marble surface with gradient backdrop",
    "text_present": false,
    "text_content": null,
    "aspect_ratio": "16:9"
  },
  "imageUrl": "https://example.com/product-image.jpg",
  "userId": "user123",
  "organizationId": "org456"
}
```

### Error Response (400/500)

```json
{
  "error": "Failed to generate prompt from image",
  "message": "imageUrl is required and must be a valid URL string",
  "details": "..."
}
```

## Features

### Detailed Prompt Generation

The API generates comprehensive prompts that include:

- **Subject details**: Main subject, pose, angle, expression
- **Visual elements**: Clothing, accessories, textures, materials, reflections
- **Color information**: Colors, gradients, patterns
- **Environment**: Background elements and setting
- **Lighting**: Setup, shadows, highlights, reflections
- **Camera specs**: Lens, aperture, depth of field, focus
- **Mood & style**: Aesthetics and emotional tone
- **Composition**: Framing and layout

### Structured Variables

Extracts key metadata into a structured JSON object:

- `subject`: Main subject of the image
- `style`: Visual style (photorealistic, illustration, 3D render, etc.)
- `lighting`: Lighting setup description
- `color_palette`: Array of dominant colors
- `mood`: Emotional tone
- `composition`: Framing and layout approach
- `camera_angle`: Perspective description
- `background`: Background description
- `text_present`: Boolean indicating if text appears
- `text_content`: Transcribed text if present
- `aspect_ratio`: Estimated aspect ratio

## Use Cases

1. **Reverse Engineering**: Analyze competitor or reference images to understand their visual composition
2. **Style Transfer**: Extract detailed prompts to recreate similar images with variations
3. **Creative Briefs**: Generate comprehensive visual descriptions for design teams
4. **Image Documentation**: Create detailed textual descriptions of visual assets
5. **Training Data**: Generate prompts for fine-tuning image generation models

## Testing

Run the test script to verify the API works correctly:

```bash
node scripts/test-image-to-prompt.js
```

## Technical Details

- **Model**: OpenAI GPT-4o with Vision
- **Timeout**: 60 seconds
- **Max Tokens**: 2000
- **Temperature**: 0.3 (for consistent, precise descriptions)
- **Image Detail**: High (for maximum accuracy)

## Service Layer

The functionality is implemented in `services/image-prompt-service.js`:

```javascript
const { generatePromptFromImage } = require('../services/image-prompt-service');

const result = await generatePromptFromImage({ 
  imageUrl: 'https://example.com/image.jpg' 
});

console.log(result.prompt);
console.log(result.variables);
```

## Prompt Template

The system prompt is stored in `prompts/image-to-prompt.txt` and can be customized to adjust the analysis style and output format.

