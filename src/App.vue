<script setup lang="ts">
    import { ref, onMounted } from 'vue';
import { ungzip } from 'pako'
import Papa from 'papaparse';

// --- Types ---
type ProcessedSegment = {
  text: string;
  type: 'found' | 'not-found' | 'unknown';
  data?: {
    chapter?: string;
    pinyin: string;
    category?: string;
    meaning: string;
  }[];
};

type Dictionary = Record<string, { 
  pinyin: string; 
  meaning: string; 
}[]>;

type CsvDictionary = Record<string, { 
  chapter: string;
  pinyin: string;
  category: string;
  meaning: string;
}[]>;

type UnknownWord = {
  index: number;
  word: string;
  pinyin?: string;
  meaning?: string;
};

// --- State ---
const csvInput = ref('');
const scriptInput = ref('');

const ccCedict = ref<Dictionary>({});
const csvDictionary = ref<CsvDictionary>({});
const processedOutput = ref<ProcessedSegment[]>([]);

const selectedSegmentDetails = ref<ProcessedSegment | null>(null);
const unknownWords = ref<UnknownWord[]>([]);

// --- Lifecycle ---
onMounted(async () => {
  try {
    // Load default CSV dictionary
    const csvResponse = await fetch('/chinese_word_highligher/default_values.csv');
    if (!csvResponse.ok) throw new Error("Failed to load CSV dictionary.");
    csvInput.value = await csvResponse.text();
    parseCsvData();

    // Load and decompress CC-CEDICT
    const dictResponse = await fetch('/chinese_word_highligher/cedict.json.gz');
    if (!dictResponse.ok) throw new Error("Failed to load CC-CEDICT.");
    const compressedData = await dictResponse.arrayBuffer();
    const decompressedData = await ungzip(new Uint8Array(compressedData));
    ccCedict.value = JSON.parse(new TextDecoder().decode(decompressedData));

  } catch (error) {
    console.error("Error loading dictionaries:", error);
    try {
        const dictResponse = await fetch('/chinese_word_highligher/cedict.json.gz');
        if (!dictResponse.ok) throw new Error("Failed to load CC-CEDICT.");
        ccCedict.value = JSON.parse(await dictResponse.text());
    } catch (error) {
    console.error("Error loading dictionaries (backup):", error);
    }
  }
});

// --- Parse CSV Dictionary ---
const parseCsvData = () => {
  console.log("Parsing CSV data...");

  const result = Papa.parse(csvInput.value.trim(), {
    header: true, 
    skipEmptyLines: true
  });

  const parsedData = result.data as Record<string, string>[];
  const newDictionary: CsvDictionary = {};
  parsedData.forEach((entry_raw) => {
      const simplified = entry_raw["Simplified"];
      if (simplified) {
          const entry = {
              chapter: entry_raw["Chapter"].trim(),
              pinyin: entry_raw["Pinyin"].trim(),
              category: entry_raw["Category"].trim(),
              meaning: entry_raw["Meaning"].trim()
          };
          if (!newDictionary[simplified]) {
            newDictionary[simplified] = [];
          }
          newDictionary[simplified].push(entry);
        }
    });

  csvDictionary.value = newDictionary;
  console.log("CSV Dictionary populated:", csvDictionary.value);
};

// --- Process Script ---
const processScript = () => {
  if (!Object.keys(ccCedict.value).length) {
    console.warn("CC-CEDICT is empty. Cannot process script.");
    processedOutput.value.push({ text: 'Error: Dictionary not loaded or empty.', type: 'not-found' });
    return;
  }

  const scriptText = scriptInput.value;
  const cedictWords = Object.keys(ccCedict.value).sort((a, b) => b.length - a.length); // Sort longest first
  const results: ProcessedSegment[] = [];
  const unknownSet = new Set<string>();

 function GetSegmentsInCsv(word: string): string[] | null {
  if (csvDictionary.value[word]) return [word]; // Direct match

  // Generate all possible segmentations
  const validSegments: string[] = [];
  for (let i = 1; i < word.length; i++) {
    const left = word.substring(0, i);
    const right = word.substring(i);

    if (csvDictionary.value[left] && csvDictionary.value[right]) {
      validSegments.push(left, right);
      return validSegments;
    }
  }

  return null; // No valid segmentation found
}

  let i = 0;
  while (i < scriptText.length) {
    if (scriptText[i] == '\n') {
      results.push({ text: '\n', type: 'unknown' });
      i += 1
      continue;
    }

    if (scriptText[i].match(/[a-z]/i)) {
      results.push({ text: scriptText[i], data: {pinyin: scriptText[i]}, type: 'found' });
      i += 1
      continue;
    }

    if (!scriptText[i].trim()) {
      i += 1;
      continue;
    }

    let matchFound = false;
    for (const word of cedictWords) {
    if (scriptText.startsWith(word, i)) {
      matchFound = true;
      const validSegments = GetSegmentsInCsv(word);

      if (validSegments) {
        // Push each segment separately
        for (const segment of validSegments) {
          results.push({
            text: segment,
            type: 'found',
            data: csvDictionary.value[segment]
          });
        }
      } else {
        results.push({
          text: word,
          type: 'not-found',
          data: ccCedict.value[word]
        });
        unknownSet.add(word); // Collect "not-found" words
      }

      i += word.length;
      break;
    }
  }

    if (!matchFound) {
      for (const word in csvDictionary.value) {
        if (scriptText.startsWith(word, i)) {
          results.push({
            text: word,
            type: 'found',
            data: csvDictionary.value[word]
          });
          i += word.length;
          matchFound = true;
          break;
        }
      }
    }

    if (!matchFound) {
      i += 1; // Move to the next character
    }
  }

  processedOutput.value = results;

  unknownWords.value = Array.from(unknownSet).map((word, index) => ({
    index: index + 1,
    word: word,
    pinyin: ccCedict.value[word]?.[0]?.pinyin || "",
    meaning: ccCedict.value[word]?.[0]?.meaning || ""
  }));
};

// --- Show Details ---
const showDetails = (segment: ProcessedSegment) => {
  if (segment.type === 'found' && segment.data?.length) {
    selectedSegmentDetails.value = segment;
  }
};

</script>

<template>
    <div class="app-container">
        <h1>Chinese Word Highlighter</h1>

        <!-- Input -->
        <div class="input-area">
            <div class="textarea-container">
                <label for="csvInput">Dictionary</label>
                <textarea id="csvInput" v-model="csvInput" rows="10" placeholder="Paste CSV data..." @change="parseCsvData"></textarea>
            </div>
            <div class="textarea-container">
                <label for="scriptInput">Script Text</label>
                <textarea id="scriptInput" v-model="scriptInput" rows="10" placeholder="Paste text..."></textarea>
            </div>
        </div>

        <!-- Button -->
        <button @click="processScript" class="process-button">Process Script</button>

        <!-- Output -->
        <div class="output-container">

            <!-- normal Highlighted text -->
            <div class="output-display">
                <template v-for="(segment, index) in processedOutput" :key="index">
                    <span v-if="segment.text !== '\n'" :class="segment.type" @click="showDetails(segment)">
                        {{ segment.text }}
                    </span>
                    <br v-else> <!-- Insert <br> for newlines -->
                </template>
            </div>

            <!-- Info -->
            <div v-if="selectedSegmentDetails?.data" class="details-box">
                <div><h3>{{ selectedSegmentDetails.text }}</h3></div>
                <div v-for="(entry, idx) in selectedSegmentDetails.data" :key="idx">
                    <div><strong>Pinyin:</strong> {{ entry.pinyin }}</div>
                    <div><strong>Chapter:</strong> {{ entry.chapter }}</div>
                    <div><strong>Category:</strong> {{ entry.category }}</div>
                    <div><strong>Meaning:</strong> {{ entry.meaning }}</div>
                    <hr v-if="idx < selectedSegmentDetails.data.length - 1">
                </div>
            </div>

            <!-- annotated -->
            <div class="output-annotated">
                <template v-for="(segment, index) in processedOutput" :key="index">
                    <span v-if="segment.text !== '\n'">
                        <ruby v-if="segment.data && segment.data.length">
                            {{ segment.text }}
                            <rt>{{ segment.data[0].pinyin }}</rt> <!-- Show pinyin above -->
                        </ruby>
                        <span v-else>{{ segment.text }}</span> <!-- Show normal text if no pinyin -->
                    </span>
                    <br v-else> <!-- Insert <br> for newlines -->
                </template>
            </div>

        </div>


        <!-- Table -->
        <div v-if="unknownWords.length > 0" class="unknown-words-table">
            <br>
            <h2>New Words</h2>
            <br>
            <table>
                <thead>
                    <tr>
                        <th>Index</th>
                        <th>Word</th>
                        <th>Pinyin</th>
                        <th>Meaning</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="word in unknownWords" :key="word.index">
                        <td>{{ word.index }}</td>
                        <td>{{ word.word }}</td>
                        <td>{{ word.pinyin }}</td>
                        <td>{{ word.meaning }}</td>
                    </tr>
                </tbody>
            </table>
        </div>

    </div>
</template>
