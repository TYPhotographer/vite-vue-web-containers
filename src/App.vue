<template>
  <div class="container">
    <div class="edit-block">
      <div class="editor">
        <textarea ref="textareaRef" @input="handleTextAreaInput" />
      </div>
      <div class="preview">
        <iframe ref="iframeRef" />
      </div>
    </div>
    <div ref="terminalRef" class="terminal"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { WebContainer } from '@webcontainer/api'
import { Terminal } from 'xterm'
import './style.css'
import 'xterm/css/xterm.css'

const files = {
  'src': {
    directory: {
      'test.txt': {
        file: {
          contents: 'Hello World',
        }
      }
    }
  },
  'index.js': {
    file: {
      contents: `
        import express from 'express';
        const app = express();
        const port = 3111;
        
        app.get('/', (req, res) => {
            res.send('Welcome to a WebContainers app! 🥳');
        });
        
        app.listen(port, () => {
            console.log(\`App is live at http://localhost:\${port}\`);
        });`,
    },
  },
  'package.json': {
    file: {
      contents: `
        {
          "name": "example-app",
          "type": "module",
          "dependencies": {
            "express": "latest",
            "nodemon": "latest"
          },
          "scripts": {
            "start": "nodemon --watch './' index.js"
          }
        }`,
    },
  },
}

let webcontainerInstance: WebContainer | null = null

const textareaRef = ref<HTMLTextAreaElement | null>(null)
const iframeRef = ref<HTMLIFrameElement | null>(null)
const terminalRef = ref<HTMLDivElement | null>(null)

function handleTextAreaInput(e: Event) {
  const textarea = e.currentTarget as HTMLTextAreaElement
  writeIndexJS(textarea.value)
}

onMounted(async () => {
  if (textareaRef.value === null) {
    throw new Error('textareaRef is null')
  }
  // 將 textarea 的值設為 index.js 的內容
  textareaRef.value.value = files['index.js'].file.contents

  const terminal = new Terminal({
    convertEol: true,
  })
  if (terminalRef.value === null) {
    throw new Error('terminalRef is null')
  }
  terminal.open(terminalRef.value)

  webcontainerInstance = await WebContainer.boot()
  
  webcontainerInstance.on('server-ready', (port, url) => {
    if (iframeRef.value === null) {
      throw new Error('iframeRef is null')
    }
    console.log('server-ready', port, url)
    iframeRef.value.src = url
  })
  webcontainerInstance.on('port', (port, action) => {
    console.log('port', port, action)
    if (iframeRef.value === null) {
      throw new Error('iframeRef is null')
    }
    if (action === 'close') {
      console.log('port open')
      iframeRef.value.src = ''
    }
  })
  webcontainerInstance.on('error', () => {
    console.log('error')
  })
  await webcontainerInstance.mount(files)

  // 讀取檔案
  const file = await webcontainerInstance.fs.readFile('/index.js')
  // 寫入檔案
  await webcontainerInstance.fs.writeFile('/index.js', "console.log('Hello World')")
  // recursive參數為true，可以支援建立多層資料夾
  await webcontainerInstance.fs.mkdir('/testFolder/childFolder', { recursive: true })
  // 讀取檔案夾 回傳一個 DirEnt[]
  const dir = await webcontainerInstance.fs.readdir('/src', { withFileTypes: true })

  // npm i
  // const exitCode = await installDependencies(terminal)
  // if (exitCode !== 0) {
  //   throw new Error('Installation failed')
  // }

  // pnpm run start
  // startDevServer(terminal)

  startShell(terminal)
})

async function startShell(terminal: Terminal) {
  if (webcontainerInstance == null) {
    throw new Error('webcontainerInstance is null')
  }
  const shellProcess = await webcontainerInstance.spawn('jsh');
  shellProcess.output.pipeTo(
    new WritableStream({
      write(data) {
        terminal.write(data);
      },
    })
  );
  const input = shellProcess.input.getWriter();
  terminal.onData((data) => {
    input.write(data);
  })
  return shellProcess;
};

async function installDependencies(terminal: Terminal) {
  if (webcontainerInstance == null) {
    throw new Error('webcontainerInstance is null')
  }

  const installProcess = await webcontainerInstance.spawn('npm', ['install'])
  installProcess.output.pipeTo(new WritableStream({
    write(data) {
      console.log(data)
    }
  }))

  // Wait for install command to exit
  return installProcess.exit
}

async function startDevServer(terminal: Terminal) {
  if (webcontainerInstance == null) {
    throw new Error('webcontainerInstance is null')
  }
  const serverProcess = await webcontainerInstance.spawn('npm', ['run', 'start'])

  serverProcess.output.pipeTo(
    new WritableStream({
      write(data) {
        terminal.write(data)
      },
    })
  )
  // Wait for `server-ready` event
  webcontainerInstance.on('server-ready', (port, url) => {
    if (iframeRef.value == null) {
      throw new Error('iframeRef is null')
    }
    iframeRef.value.src = url
  })
}

/**
 * @param {string} content
 */

async function writeIndexJS(content: string) {
  if (webcontainerInstance === null) {
    throw new Error('webcontainerInstance is null')
  }
  await webcontainerInstance.fs.writeFile('/index.js', content)
}

</script>