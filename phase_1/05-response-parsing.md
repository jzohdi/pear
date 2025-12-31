# Response Parsing

**Document**: `05-response-parsing.md`  
**Purpose**: Specification for parsing and extracting data from LLM responses

---

## Overview

The Response Parser extracts structured data from LLM responses, specifically:
- Code blocks with file paths
- Artifacts (files to create/modify)
- Markdown sections

This is critical because LLM output is free-form text, but we need structured data to create files and update state.

---

## Parser Interface

```typescript
interface ResponseParser {
  /**
   * Extract all code blocks from a response
   */
  parseCodeBlocks(response: string): ParsedCodeBlock[];
  
  /**
   * Extract file artifacts from code blocks
   */
  parseArtifacts(response: string): Artifact[];
  
  /**
   * Extract markdown sections by heading
   */
  parseSections(response: string): ParsedSection[];
  
  /**
   * Basic TypeScript syntax validation
   */
  validateTypeScript(code: string): ValidationResult;
}
```

---

## Code Block Extraction

### Format

LLM outputs code in fenced code blocks:

````markdown
```typescript
// src/features/auth/types.ts
export interface User {
  id: string;
}
```
````

### Parsing Rules

1. **Match fenced blocks**: Look for triple backticks
2. **Extract language**: From the opening fence (e.g., `typescript`)
3. **Extract file path**: From the first line comment
4. **Extract content**: Everything between fences (excluding the path comment)

### File Path Patterns

The parser should recognize these patterns for file paths:

```typescript
const FILE_PATH_PATTERNS = [
  // Standard: // path/to/file.ext
  /^\/\/ ([\w\-./]+\.(ts|tsx|js|jsx|md|json|yaml))$/m,
  
  // With "File:" prefix: // File: path/to/file.ext
  /^\/\/ File: ([\w\-./]+\.(ts|tsx|js|jsx|md|json|yaml))$/m,
  
  // Markdown heading for .md files: # filename.md
  /^# ([\w\-./]+\.md)$/m,
];
```

### Implementation

```typescript
interface ParsedCodeBlock {
  /** Language from fence (e.g., "typescript", "markdown") */
  language: string;
  
  /** File path if found (null otherwise) */
  filePath: string | null;
  
  /** The code content (without path comment) */
  content: string;
  
  /** Line number where block starts in original */
  startLine: number;
  
  /** Line number where block ends */
  endLine: number;
}

function parseCodeBlocks(response: string): ParsedCodeBlock[] {
  const blocks: ParsedCodeBlock[] = [];
  const lines = response.split('\n');
  
  let inBlock = false;
  let currentBlock: Partial<ParsedCodeBlock> | null = null;
  let blockContent: string[] = [];
  let blockStartLine = 0;
  
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    
    // Check for opening fence
    if (!inBlock && line.match(/^```(\w+)?$/)) {
      inBlock = true;
      blockStartLine = i;
      const match = line.match(/^```(\w+)?$/);
      currentBlock = {
        language: match?.[1] || 'text',
        startLine: i,
      };
      blockContent = [];
      continue;
    }
    
    // Check for closing fence
    if (inBlock && line === '```') {
      inBlock = false;
      
      // Extract file path from first line if present
      const { filePath, content } = extractFilePath(blockContent);
      
      blocks.push({
        language: currentBlock!.language!,
        filePath,
        content,
        startLine: blockStartLine,
        endLine: i,
      });
      
      currentBlock = null;
      continue;
    }
    
    // Accumulate block content
    if (inBlock) {
      blockContent.push(line);
    }
  }
  
  return blocks;
}

function extractFilePath(lines: string[]): { filePath: string | null; content: string } {
  if (lines.length === 0) {
    return { filePath: null, content: '' };
  }
  
  const firstLine = lines[0];
  
  // Try each pattern
  for (const pattern of FILE_PATH_PATTERNS) {
    const match = firstLine.match(pattern);
    if (match) {
      return {
        filePath: match[1],
        content: lines.slice(1).join('\n'),
      };
    }
  }
  
  // No file path found
  return {
    filePath: null,
    content: lines.join('\n'),
  };
}
```

---

## Artifact Extraction

### Purpose

Convert parsed code blocks into file artifacts that can be written to disk.

### Implementation

```typescript
interface Artifact {
  path: string;
  content: string;
  action: 'create' | 'modify' | 'delete';
}

function parseArtifacts(response: string): Artifact[] {
  const codeBlocks = parseCodeBlocks(response);
  
  const artifacts: Artifact[] = [];
  const pathToContent = new Map<string, string[]>();
  
  for (const block of codeBlocks) {
    if (block.filePath) {
      // Group content by file path (multiple blocks for same file)
      const existing = pathToContent.get(block.filePath) || [];
      existing.push(block.content);
      pathToContent.set(block.filePath, existing);
    }
  }
  
  for (const [path, contents] of pathToContent) {
    artifacts.push({
      path,
      content: contents.join('\n\n'),
      action: 'create', // Default action; could detect 'modify' with context
    });
  }
  
  return artifacts;
}
```

### Handling Multiple Blocks for Same File

When LLM outputs multiple code blocks for the same file:

1. **Test files**: Often have multiple test implementations
2. **Appending**: Join with double newline
3. **Conflict detection**: Could warn if content seems inconsistent

```typescript
// Example: Multiple test implementations for same file
// Block 1
// src/features/auth/__tests__/auth.test.ts
// Test: should return valid URL
it('should return valid URL', () => { ... });

// Block 2  
// src/features/auth/__tests__/auth.test.ts
// Test: should include state
it('should include state', () => { ... });

// Result: Both tests concatenated into one artifact
```

---

## Section Extraction

### Purpose

Extract markdown sections for structured reading (e.g., "Implementation notes").

### Implementation

```typescript
interface ParsedSection {
  /** The heading text (without #) */
  heading: string;
  
  /** Heading level (1 = #, 2 = ##, etc.) */
  level: number;
  
  /** Content under this heading (until next heading) */
  content: string;
}

function parseSections(response: string): ParsedSection[] {
  const sections: ParsedSection[] = [];
  const lines = response.split('\n');
  
  let currentSection: ParsedSection | null = null;
  let contentLines: string[] = [];
  
  for (const line of lines) {
    const headingMatch = line.match(/^(#{1,6})\s+(.+)$/);
    
    if (headingMatch) {
      // Save previous section
      if (currentSection) {
        currentSection.content = contentLines.join('\n').trim();
        sections.push(currentSection);
      }
      
      // Start new section
      currentSection = {
        level: headingMatch[1].length,
        heading: headingMatch[2],
        content: '',
      };
      contentLines = [];
    } else if (currentSection) {
      contentLines.push(line);
    }
  }
  
  // Save final section
  if (currentSection) {
    currentSection.content = contentLines.join('\n').trim();
    sections.push(currentSection);
  }
  
  return sections;
}
```

### Finding Specific Sections

```typescript
function findSection(response: string, heading: string): string | null {
  const sections = parseSections(response);
  const section = sections.find(
    s => s.heading.toLowerCase() === heading.toLowerCase()
  );
  return section?.content ?? null;
}

// Usage
const implementationNotes = findSection(response, 'Implementation notes');
const suggestedOrder = findSection(response, 'Suggested Order');
```

---

## Validation

### TypeScript Validation

Basic syntax checking without full compilation:

```typescript
interface ValidationResult {
  valid: boolean;
  errors: string[];
}

function validateTypeScript(code: string): ValidationResult {
  const errors: string[] = [];
  
  // Check for balanced braces
  let braceCount = 0;
  let parenCount = 0;
  let bracketCount = 0;
  
  for (const char of code) {
    switch (char) {
      case '{': braceCount++; break;
      case '}': braceCount--; break;
      case '(': parenCount++; break;
      case ')': parenCount--; break;
      case '[': bracketCount++; break;
      case ']': bracketCount--; break;
    }
    
    if (braceCount < 0 || parenCount < 0 || bracketCount < 0) {
      errors.push('Unbalanced brackets detected');
      break;
    }
  }
  
  if (braceCount !== 0) errors.push(`Unbalanced braces: ${braceCount}`);
  if (parenCount !== 0) errors.push(`Unbalanced parentheses: ${parenCount}`);
  if (bracketCount !== 0) errors.push(`Unbalanced brackets: ${bracketCount}`);
  
  // Check for common syntax errors
  if (code.includes('export export')) {
    errors.push('Duplicate export keyword');
  }
  
  if (code.match(/function\s+\(/)) {
    errors.push('Function missing name');
  }
  
  return {
    valid: errors.length === 0,
    errors,
  };
}
```

**Note**: Full TypeScript validation would require `ts-morph` or the TypeScript compiler API. For Phase 1, basic checks are sufficient.

---

## Edge Cases

### Empty Code Blocks

```typescript
// Handle gracefully
if (block.content.trim() === '') {
  // Skip empty blocks or warn
  console.warn(`Empty code block at line ${block.startLine}`);
}
```

### Missing File Path

When code block has no file path:

```typescript
if (!block.filePath) {
  // Option 1: Ask user
  const path = await cli.promptForFreeformInput(
    'Where should this code be saved?'
  );
  
  // Option 2: Infer from language
  if (block.language === 'typescript') {
    // Use context to guess (e.g., current unit being implemented)
  }
  
  // Option 3: Skip and warn
  console.warn('Code block without file path, skipping');
}
```

### Nested Code Blocks

Rare, but possible in markdown explanations:

````markdown
Here's how to write a code block:

```markdown
```typescript
const x = 1;
```
```
````

Handle by tracking nesting depth:

```typescript
// Simplified: don't parse code blocks inside markdown code blocks
if (block.language === 'markdown') {
  // Return content as-is, don't parse nested blocks
}
```

### Code Block in Implementation Notes

Sometimes the LLM includes explanatory code that shouldn't be artifacts:

```markdown
**Implementation notes:**
- We use `Map<string, number>` for O(1) lookups:
  ```typescript
  const cache = new Map<string, number>();
  ```
```

Solution: Only extract artifacts from top-level code blocks, not those in sections like "Implementation notes" or "Explanation".

```typescript
function parseArtifacts(response: string): Artifact[] {
  const codeBlocks = parseCodeBlocks(response);
  const sections = parseSections(response);
  
  // Get line ranges of "non-artifact" sections
  const nonArtifactSections = ['implementation notes', 'explanation', 'notes'];
  const excludedRanges: Array<[number, number]> = [];
  
  // ... calculate excluded ranges from sections
  
  return codeBlocks
    .filter(block => {
      // Exclude blocks inside non-artifact sections
      return !excludedRanges.some(
        ([start, end]) => block.startLine >= start && block.endLine <= end
      );
    })
    .filter(block => block.filePath !== null)
    .map(block => ({
      path: block.filePath!,
      content: block.content,
      action: 'create' as const,
    }));
}
```

---

## Testing the Parser

```typescript
// tests/llm/parser.test.ts

describe('ResponseParser', () => {
  describe('parseCodeBlocks', () => {
    it('should extract code block with file path', () => {
      const response = `
Here's the implementation:

\`\`\`typescript
// src/utils/helper.ts
export function helper() {
  return true;
}
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks).toHaveLength(1);
      expect(blocks[0].language).toBe('typescript');
      expect(blocks[0].filePath).toBe('src/utils/helper.ts');
      expect(blocks[0].content).toContain('export function helper');
    });
    
    it('should handle code block without file path', () => {
      const response = `
\`\`\`typescript
const x = 1;
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks[0].filePath).toBeNull();
      expect(blocks[0].content).toBe('const x = 1;');
    });
    
    it('should handle multiple code blocks', () => {
      const response = `
\`\`\`typescript
// src/a.ts
const a = 1;
\`\`\`

\`\`\`typescript
// src/b.ts
const b = 2;
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks).toHaveLength(2);
      expect(blocks[0].filePath).toBe('src/a.ts');
      expect(blocks[1].filePath).toBe('src/b.ts');
    });
  });
  
  describe('parseArtifacts', () => {
    it('should combine multiple blocks for same file', () => {
      const response = `
\`\`\`typescript
// src/test.ts
// Test 1
it('test 1', () => {});
\`\`\`

\`\`\`typescript
// src/test.ts
// Test 2
it('test 2', () => {});
\`\`\`
      `;
      
      const artifacts = parseArtifacts(response);
      
      expect(artifacts).toHaveLength(1);
      expect(artifacts[0].content).toContain('test 1');
      expect(artifacts[0].content).toContain('test 2');
    });
  });
});
```

---

## Related Documents

- [04-llm-prompts.md](./04-llm-prompts.md) — Output format specifications in prompts
- [02-components.md](./02-components.md) — Response Parser component
- [06-context-management.md](./06-context-management.md) — Context for LLM requests

