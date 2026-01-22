---
name: docusaurus-conventions
description: Best practices and conventions for Docusaurus documentation sites. Use when creating pages, components, configuring sidebars, writing MDX content, or customizing Docusaurus themes for the Physical AI textbook.
---

# Docusaurus Conventions for Physical AI Textbook

## File Structure

The project follows this organization:

```
physical-ai-textbook/
├── docs/                          # All documentation content
│   ├── intro.md                   # Landing page (sidebar_position: 1)
│   ├── module-1-ros2/
│   │   ├── _category_.json        # Sidebar category configuration
│   │   ├── index.md               # Module overview
│   │   ├── nodes-topics-services.md
│   │   ├── python-rclpy.md
│   │   └── urdf-humanoids.md
│   ├── module-2-digital-twin/
│   ├── module-3-nvidia-isaac/
│   └── module-4-vla/
├── src/
│   ├── components/                # Reusable React components
│   │   └── Chatbot/              # RAG chatbot component
│   ├── pages/                     # Custom pages (not in docs)
│   └── css/
│       └── custom.css            # Global style overrides
├── static/                        # Static assets (images, files)
├── docusaurus.config.js          # Main configuration
├── sidebars.js                   # Sidebar navigation structure
└── tailwind.config.js            # Tailwind CSS configuration
```

## Frontmatter Standards

Every markdown file in `docs/` MUST have this frontmatter:

```yaml
---
sidebar_position: 1
title: "Human-Readable Page Title"
description: "SEO description for search engines (max 160 characters)"
keywords: [physical ai, ros2, humanoid robotics]
---
```

## Category Configuration

Each module folder needs a `_category_.json` file:

```json
{
  "label": "Module 1: ROS 2 Fundamentals",
  "position": 2,
  "collapsed": false,
  "link": {
    "type": "generated-index",
    "description": "Learn the robotic nervous system with ROS 2"
  }
}
```

## MDX Features to Use

### Admonitions (Callout Boxes)

Use these callout boxes to highlight important information:

```mdx
:::tip Pro Tip
This is helpful advice for the reader.
:::

:::warning Caution
This warns about potential issues.
:::

:::danger Critical
This is a critical warning that must not be ignored.
:::

:::info Background
This provides additional context.
:::
```

### Code Blocks with Titles and Line Highlighting

Always add titles to code blocks and highlight important lines:

````mdx
```python title="ros2_node_example.py" showLineNumbers
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
        # highlight-next-line
        self.get_logger().info('Node started!')
```
````

### Tabs for Multiple Platforms/Languages

Use tabs when showing platform-specific instructions:

```mdx
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
  <TabItem value="ubuntu" label="Ubuntu" default>
    ```bash
    sudo apt install ros-humble-desktop
    ```
  </TabItem>
  <TabItem value="docker" label="Docker">
    ```bash
    docker pull ros:humble
    ```
  </TabItem>
</Tabs>
```

### Importing Custom Components

Import and use React components in MDX files:

```mdx
import { Chatbot } from '@site/src/components/Chatbot';
import InteractiveDemo from '@site/src/components/InteractiveDemo';

<Chatbot />
```

## Sidebar Configuration (sidebars.js)

Structure your sidebar navigation like this:

```javascript
module.exports = {
  docs: [
    'intro',
    {
      type: 'category',
      label: 'Module 1: ROS 2',
      items: [
        'module-1-ros2/index',
        'module-1-ros2/nodes-topics-services',
        'module-1-ros2/python-rclpy',
        'module-1-ros2/urdf-humanoids',
      ],
    },
    {
      type: 'category',
      label: 'Module 2: Digital Twin',
      items: [
        'module-2-digital-twin/index',
        'module-2-digital-twin/gazebo-physics',
        'module-2-digital-twin/unity-rendering',
        'module-2-digital-twin/sensors-simulation',
      ],
    },
    {
      type: 'category',
      label: 'Module 3: NVIDIA Isaac',
      items: [
        'module-3-nvidia-isaac/index',
        'module-3-nvidia-isaac/isaac-sim',
        'module-3-nvidia-isaac/isaac-ros-vslam',
        'module-3-nvidia-isaac/nav2-path-planning',
      ],
    },
    {
      type: 'category',
      label: 'Module 4: Vision-Language-Action',
      items: [
        'module-4-vla/index',
        'module-4-vla/voice-to-action',
        'module-4-vla/cognitive-planning-llms',
        'module-4-vla/capstone-autonomous-humanoid',
      ],
    },
  ],
};
```

## Component Standards

Place reusable components in `src/components/` with this structure:

```
src/components/Chatbot/
├── index.tsx           # Main export
├── ChatMessage.tsx     # Individual message component
├── ChatInput.tsx       # Input field component
├── ChatHistory.tsx     # Message list component
└── types.ts            # TypeScript interfaces
```

### Component Template

Every component should follow this pattern:

```tsx
import React from 'react';

/**
 * ChatMessage displays a single message in the chat interface
 * @param content - The message text content
 * @param isUser - Whether this message is from the user (true) or AI (false)
 * @param timestamp - Optional timestamp for the message
 */
interface ChatMessageProps {
  content: string;
  isUser: boolean;
  timestamp?: Date;
}

export const ChatMessage: React.FC<ChatMessageProps> = ({ 
  content, 
  isUser, 
  timestamp 
}) => {
  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'} mb-4`}>
      <div
        className={`max-w-[80%] rounded-lg px-4 py-2 ${
          isUser
            ? 'bg-blue-600 text-white'
            : 'bg-gray-100 text-gray-900 dark:bg-gray-800 dark:text-gray-100'
        }`}
      >
        <p className="text-sm">{content}</p>
        {timestamp && (
          <span className="text-xs opacity-70">
            {timestamp.toLocaleTimeString()}
          </span>
        )}
      </div>
    </div>
  );
};
```

## Styling with Tailwind CSS

Use Tailwind utility classes consistently. Here are common patterns for this project:

```jsx
// Card component
<div className="rounded-lg border border-gray-200 p-6 shadow-sm hover:shadow-md transition-shadow">

// Code block wrapper
<div className="bg-gray-900 rounded-lg overflow-hidden">

// Responsive container
<div className="max-w-4xl mx-auto px-4 sm:px-6 lg:px-8">

// Button styles
<button className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors">

// Dark mode aware text
<p className="text-gray-900 dark:text-gray-100">
```

## Links and References

### Internal Links

Link to other documentation pages:

```mdx
See [ROS 2 Nodes](/docs/module-1-ros2/nodes-topics-services) for more details.

Learn about [simulation basics](./gazebo-physics.md) first.
```

### Asset References

Reference images and static files:

```mdx
![Robot Architecture](/img/robot-architecture.png)

Download the [example code](/files/ros2-example.zip).
```

## Best Practices

1. **One concept per page**: Each markdown file should focus on a single topic
2. **Progressive disclosure**: Start simple, add complexity gradually
3. **Code first**: Show working code, then explain it
4. **Visual aids**: Include diagrams for complex architectures
5. **Cross-references**: Link related topics to help navigation
6. **Consistent terminology**: Use terms defined in physical-ai-terminology skill
