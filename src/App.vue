<template>
  <div class="container">
    <h1>Conversor de Formatos</h1>

    <div class="controls">
      <label>Formato de entrada:</label>
      <select v-model="inputFormat">
        <option value="json">JSON</option>
        <option value="xml">XML</option>
        <option value="csv">CSV</option>
        <option value="yaml">YAML</option>
      </select>

      <label>Formato de salida:</label>
      <select v-model="outputFormat">
        <option value="json">JSON</option>
        <option value="xml">XML</option>
        <option value="csv">CSV</option>
        <option value="yaml">YAML</option>
      </select>
    </div>

    <textarea v-model="inputText" placeholder="Pegá aquí tu contenido de entrada" rows="10"></textarea>

    <button @click="convert">Convertir</button>

    <textarea v-model="outputText" readonly placeholder="Resultado de salida" rows="10"></textarea>
  </div>
  <hr>

<h2>Conversor de XML a cURL para Postman</h2>

<textarea id="xmlCurlInput" placeholder="Pegá aquí tu XML para generar cURL" rows="15" cols="80" style="width: 100%;"></textarea>
<br><br>
<button @click="convertirXmlACurl">Generar cURL</button>

<br><br>
<textarea id="curlOutput" placeholder="Aquí aparecerá tu cURL listo para Postman" rows="20" cols="80" style="width: 100%;" readonly></textarea>

</template>

<script setup>
import { ref } from 'vue';
import { XMLParser, XMLBuilder } from 'fast-xml-parser';
import Papa from 'papaparse';
import yaml from 'js-yaml';

const inputText = ref('');
const outputText = ref('');
const inputFormat = ref('json');
const outputFormat = ref('xml');

// Opciones para reconocer arrays en XML
const parserOptions = {
  ignoreAttributes: false,
  attributeNamePrefix: "@_",
  isArray: (tagName, jPath, isLeafNode, isAttribute) => {
    // Considerar array si el tag se repite (típico caso XML)
    return ['Item', 'element', 'row'].includes(tagName);
  }
};

const builderOptions = {
  ignoreAttributes: false,
  attributeNamePrefix: "@_"
};
// Limpia estructuras tipo { "productos": { "item": [ ... ] } } y las convierte en { "productos": [ ... ] }
const cleanWrapper = (obj, seenKeys = new Map()) => {
  if (Array.isArray(obj)) {
    return obj.map((item) => cleanWrapper(item, seenKeys));
  } else if (obj !== null && typeof obj === 'object') {
    const keys = Object.keys(obj);

    // Caso especial: { "Item": { ... } } → lo pasamos a [ { ... } ]
    if (keys.length === 1 && keys[0] === 'Item') {
      const val = obj['Item'];
      if (Array.isArray(val)) return val.map(v => cleanWrapper(v, seenKeys));
      else return [cleanWrapper(val, seenKeys)];
    }

    const newObj = {};
    for (const key of keys) {
      let value = obj[key];

      if (!seenKeys.has(key) && value !== '') {
        seenKeys.set(key, typeof value);
      }

      if (
        value === '' &&
        (seenKeys.get(key) === 'object' || seenKeys.get(key) === 'array')
      ) {
        value = [];
      }

      newObj[key] = cleanWrapper(value, seenKeys);
    }
    return newObj;
  }

  return obj;
};



const convert = () => {
  try {
    let result = '';
    const parser = new XMLParser(parserOptions);
    const builder = new XMLBuilder(builderOptions);

    if (inputFormat.value === 'json' && outputFormat.value === 'xml') {
      result = builder.build(JSON.parse(inputText.value));
    } else if (inputFormat.value === 'xml' && outputFormat.value === 'json') {
      const rawJson = parser.parse(inputText.value);
      const json = cleanWrapper(rawJson);
      result = JSON.stringify(json, null, 2);
    } else if (inputFormat.value === 'csv' && outputFormat.value === 'json') {
      result = JSON.stringify(Papa.parse(inputText.value, { header: true }).data, null, 2);
    } else if (inputFormat.value === 'json' && outputFormat.value === 'csv') {
      result = Papa.unparse(JSON.parse(inputText.value));
    } else if (inputFormat.value === 'yaml' && outputFormat.value === 'json') {
      result = JSON.stringify(yaml.load(inputText.value), null, 2);
    } else if (inputFormat.value === 'json' && outputFormat.value === 'yaml') {
      result = yaml.dump(JSON.parse(inputText.value));
    } else {
      result = 'Conversión no soportada aún.';
    }

    outputText.value = result;
  } catch (error) {
    outputText.value = 'Error en la conversión: ' + error.message;
  }
};

const convertirXmlACurl = () => {
  const xmlString = document.getElementById('xmlCurlInput').value;

    if (!xmlString.trim()) {
        alert('Por favor, pegá un XML válido.');
        return;
    }

    try {
        const parser = new DOMParser();
        const xmlDoc = parser.parseFromString('<root>' + xmlString + '</root>', "application/xml");

        // Validar si el XML tiene errores
        if (xmlDoc.getElementsByTagName('parsererror').length > 0) {
            alert('El XML ingresado es inválido.');
            return;
        }

        // Buscar Header
        const headerNode = xmlDoc.querySelector('Header');
        if (!headerNode) {
            alert('No se encontró la sección <Header> en el XML.');
            return;
        }

        const headers = {};
        headerNode.querySelectorAll('*').forEach(el => {
            headers[el.nodeName] = el.textContent;
        });

        // Buscar JSON
        const jsonNode = xmlDoc.querySelector('JSON');
        if (!jsonNode) {
            alert('No se encontró la sección <JSON> en el XML.');
            return;
        }

        const jsonData = xmlToJson(jsonNode);
        const cleanJsonData = cleanWrapper(jsonData);

        // Armar cURL
        const urlMatch = headers['X-Original-HTTP-Command']?.match(/POST\s+(.*?)\s+HTTP\/1\.1/);
        const url = urlMatch ? urlMatch[1] : 'URL_NO_ENCONTRADA';

        let curl = `curl -X POST "${url}" \\\n`;

        for (let key in headers) {
            if (key !== 'X-Original-HTTP-Command') {
                curl += `  -H "${key}: ${headers[key]}" \\\n`;
            }
        }

        curl += `  -d '${JSON.stringify(cleanJsonData.Data.claim, null, 2)}'`; // Puede ajustarse el nivel que quieras

        document.getElementById('curlOutput').value = curl;

    } catch (error) {
        console.error(error);
        alert('Error procesando el XML.');
    }
}

// Esta función ya la usamos antes para convertir XML a JSON
function xmlToJson(xml) {
    let obj = {};
    if (!xml) return obj;

    if (xml.children.length > 0) {
        for (let i = 0; i < xml.children.length; i++) {
            const child = xml.children[i];
            const value = xmlToJson(child);

            if (obj[child.nodeName] === undefined) {
                obj[child.nodeName] = value;
            } else {
                if (!Array.isArray(obj[child.nodeName])) {
                    obj[child.nodeName] = [obj[child.nodeName]];
                }
                obj[child.nodeName].push(value);
            }
        }
    } else {
        obj = xml.textContent.trim();
        if (obj === 'true') obj = true;
        else if (obj === 'false') obj = false;
        else if (!isNaN(obj)) obj = Number(obj);
    }
    return obj;
};

</script>



<style scoped>
.container {
  max-width: 800px;
  margin: 30px auto;
  padding: 20px;
  font-family: sans-serif;
}
textarea {
  width: 100%;
  margin: 10px 0;
  padding: 10px;
  font-family: monospace;
  resize: vertical;
}
.controls {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
  margin-bottom: 10px;
}
select {
  padding: 6px;
}
button {
  padding: 10px 20px;
  margin-bottom: 10px;
  background-color: #2d8cf0;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
button:hover {
  background-color: #1a75d1;
}
</style>
