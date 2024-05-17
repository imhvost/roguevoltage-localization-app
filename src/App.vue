<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { useFetch } from '@vueuse/core';
import { languages } from '@/languages';
import JSZip from 'jszip';

const locFile = 'localization.xml';

const files = [
  'keywords.xml',
  'localization_advice.xml',
  'localization_camp.xml',
  'localization_camp_tentative.xml',
  'localization_chapters_additional.xml',
  'localization_chapters_core.xml',
  'localization_chapters_tentative.xml',
  'localization_enemies_demo.xml',
  'localization_enemies_full.xml',
  'localization_enemies_tentative.xml',
  'localization_happenings.xml',
  'localization_happenings_tentative.xml',
  'localization_input.xml',
  'localization_messages.xml',
  'localization_modules_additional.xml',
  'localization_modules_core.xml',
  'localization_modules_properties_core.xml',
  'localization_modules_tentative.xml',
  'localization_naturaldisasters_additional.xml',
  'localization_naturaldisasters_core.xml',
  'localization_naturaldisasters_tentative.xml',
  'localization_playercharacters_additional.xml',
  'localization_playercharacters_core.xml',
  'localization_settings.xml',
  'localization_skills_additional.xml',
  'localization_skills_core.xml',
  'localization_skills_tentative.xml',
  'localization_status.xml',
  'localization_status_additional.xml',
  'localization_steampage.xml',
  'localization_store.xml',
  'localization_story_additional.xml',
  'localization_story_core.xml',
  'localization_story_tentative.xml',
  'localization_tutorial.xml',
  'localization_ui.xml',
  'localization_ui_additional.xml',
];

type Translation = {
  link: string;
  file: string;
  id: string;
  en: string;
  [key: string]: string;
};

const lang = ref<string>('');

const process = ref(false);

const json = ref<Translation[]>([]);
const strings = ref<string[]>([]);
const keywords = ref<string[]>([]);

const xmls = ref<Record<string, Blob>>({});
const parser = new DOMParser();
const serializer = new XMLSerializer();

const fixTranslation = (translation: Translation) => {
  const parameterRegex = /\$[a-z]+\$/g;
  const enParameters = translation.en.match(parameterRegex);
  if (enParameters) {
    let corrected = translation[lang.value];
    enParameters.map(parameter => {
      const regex = new RegExp(`\\${parameter}`, 'g');
      if (!corrected.match(regex)) {
        corrected = corrected.replace(parameterRegex, parameter);
      }
    });
    translation[lang.value] = corrected;
  }
};

const downloadBlob = (blob: Blob, fileName: string) => {
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = fileName;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};

const download = () => {
  if (lang.value.length < 2) {
    alert('Enter the lang');
    return;
  }
  if (process.value) {
    return;
  }
  process.value = true;

  document.querySelectorAll('.strings div').forEach((el, index) => {
    let str = el.textContent || '';
    const jsonObj = json.value[index];
    jsonObj[lang.value] = str.trim();
  });
  json.value.map(el => {
    fixTranslation(el);
    if (el.en && !el[lang.value]) {
      el[lang.value] = el.en;
    }
  });

  const blob = new Blob([JSON.stringify(json.value, null, 2)], {
    type: 'application/json',
  });
  downloadBlob(blob, `${lang.value}.json`);

  process.value = false;
};

const appendString = (set: Element | null, textContent: string) => {
  if (!set) {
    return;
  }
  const existedString = set.querySelector(`string[lang="${lang.value}"]`);
  if (existedString) {
    existedString.textContent = textContent;
    return;
  }
  const string = document.createElement('string');
  string.setAttribute('lang', lang.value);
  string.textContent = textContent;
  const tabNode = document.createTextNode('\t\t');
  const nlNode = document.createTextNode('\n');
  set.appendChild(tabNode);
  set.appendChild(string);
  set.appendChild(nlNode);
};

const createBlobXML = (doc: Document) => {
  let docString = serializer.serializeToString(doc);
  docString = docString.replace(/\s+xmlns(:\w+)?="[^"]*"/g, '');
  return new Blob([docString], { type: 'application/xml' });
};

const upload = async ($event: Event) => {
  const input = $event.currentTarget as HTMLInputElement;
  if (lang.value.length < 2) {
    input.value = '';
    alert('Enter the lang');
    return;
  }
  if (process.value) {
    return;
  }
  process.value = true;

  const filesList = input.files;
  if (!filesList || !filesList.length) {
    process.value = false;
    return;
  }
  const file = filesList[0];
  const extension = file.name.split('.')[1];
  if (extension.toLowerCase() !== 'json') {
    input.value = '';
    alert('This is not .json file');
    process.value = false;
    return;
  }

  const reader = new FileReader();
  reader.onload = async e => {
    const target = e.target;

    if (!target) {
      input.value = '';
      alert('FileReader error');
      process.value = false;
      return;
    }

    try {
      const obj: Translation[] = JSON.parse(target.result as string);

      if (!Array.isArray(obj)) {
        input.value = '';
        alert('JSON file error');
        process.value = false;
        return;
      }

      xmls.value = {};

      const { data: locFileData } = await useFetch(
        `/roguevoltage-localization/${locFile}`,
      )
        .get()
        .text();
      const locFileDoc: Document = parser.parseFromString(
        String(locFileData.value),
        'application/xml',
      );
      appendString(locFileDoc.querySelector('set'), '');

      xmls.value[locFile] = createBlobXML(locFileDoc);

      for await (const fileName of files) {
        const { data } = await useFetch(
          `/roguevoltage-localization/${fileName}`,
        )
          .get()
          .text();
        const doc: Document = parser.parseFromString(
          String(data.value),
          'application/xml',
        );
        const sets = doc.documentElement.querySelectorAll('set');
        sets.forEach(set => {
          const traslatedObj = obj.find(
            o => o.file === fileName && o.id === set.id,
          );
          if (!traslatedObj) {
            console.error(fileName, set.id);
            return;
          }
          if (!traslatedObj[lang.value]) {
            console.warn(fileName, set.id);
            return;
          }
          appendString(set, traslatedObj[lang.value]);
        });
        xmls.value[fileName] = createBlobXML(doc);
      }

      const zip = new JSZip();

      Object.keys(xmls.value).forEach(key => {
        zip.file(key, xmls.value[key]);
      });
      const archive = await zip.generateAsync({ type: 'blob' });
      downloadBlob(archive, 'roguevoltage-localization.zip');
    } catch (error) {
      alert('Error parsing JSON');
      process.value = false;
    }
  };
  reader.readAsText(file);

  input.value = '';
  process.value = false;
};

onMounted(async () => {
  for await (const fileName of files) {
    const { data } = await useFetch(`/roguevoltage-localization/${fileName}`)
      .get()
      .text();
    const doc: Document = parser.parseFromString(
      String(data.value),
      'application/xml',
    );
    const sets = doc.documentElement.querySelectorAll('set');
    sets.forEach(set => {
      const string = set.querySelector('string[lang="en"]');

      let en = string?.innerHTML || '';

      json.value.push({
        link: `https://github.com/roguevoltage/localization/blob/main/${fileName}`,
        file: `${fileName}`,
        id: `${set.id}`,
        en: en,
      });

      en = string?.textContent || '';

      if (fileName === 'keywords.xml') {
        keywords.value.push(en);
        return;
      }

      if (en) {
        en = en.replace(
          /\$(.*?)\$/g,
          '<span class="notranslate"> $$$1$$ </span>',
        );
      }
      strings.value.push(en);
    });
  }
});
</script>

<template>
  <main>
    <div class="strings">
      <div
        v-for="(item, index) in keywords"
        :key="index"
      >
        {{ item }}
      </div>
      <div
        v-for="(item, index) in strings"
        :key="index"
        v-html="item"
      ></div>
    </div>
    <div class="btns">
      <div class="languages-wrapp">
        <div class="languages-title">Enter or select lang</div>
        <div class="languages">
          <input
            type="text"
            v-model="lang"
            placeholder="Enter lang"
          />
          <select v-model="lang">
            <option
              value=""
              disabled
              selected
            >
              Select lang
            </option>
            <option
              :value="item"
              v-for="item in languages"
              :key="item"
            >
              &nbsp;&nbsp;{{ item }}&nbsp;&nbsp;
            </option>
          </select>
        </div>
      </div>
      <button
        @click="download"
        :disabled="lang.length < 2"
      >
        Download <b v-if="lang.length >= 2">{{ lang }}.json</b>
      </button>
      <label
        class="upload"
        aria-label="Upload file"
      >
        <span
          >Upload <b>.json</b> file<br />
          to create <b>.xml</b></span
        >
        <input
          @change="upload"
          type="file"
        />
      </label>
    </div>
  </main>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  min-width: 480px;
}

main {
  display: grid;
  grid-template-columns: 1fr 280px;
  gap: 12px;
  height: 100dvh;
  font-family:
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    'Open Sans',
    'Helvetica Neue',
    sans-serif;
  font-size: 14px;
}

.strings {
  display: flex;
  flex-direction: column;
  overflow: auto;
  white-space: nowrap;
}

.btns {
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.languages {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px;
  input,
  select {
    height: 32px;
    min-width: 0;
    font: inherit;
  }
}

.lang-input {
  width: 100%;
  height: 32px;
  padding: 0 12px;
}

button {
  cursor: pointer;
  padding: 8px 12px;
  font: inherit;
  &[disabled] {
    pointer-events: none;
  }
}

.upload {
  margin-top: auto;
  padding: 24px 12px;
  border: dashed 2px;
  position: relative;
  text-align: center;
  input {
    position: absolute;
    left: 0;
    top: 0;
    border-radius: inherit;
    cursor: pointer;
    opacity: 0;
    font-size: 0;
    width: 100%;
    height: 100%;
    z-index: 6;
    &::-webkit-file-upload-button,
    &::file-selector-button {
      display: none;
      appearance: none;
    }
  }
}
</style>
