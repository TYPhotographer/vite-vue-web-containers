<template>
  <div class="container">
    <div class="edit-block">
      <div class="tree">
        <FolderTree :list="currentDirs" @click-node="clickEditNode" />
      </div>
      <div class="preview">
        <iframe ref="iframeRef" />
      </div>
    </div>
    <div class="editor">
        <textarea ref="textareaRef" @input="handleTextAreaInput" />
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
import FolderTree from './components/FolderTree.vue'

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

const currentEditNode = ref<DirSys | null>(null)
const currentDirs = ref<DirSys[]>([])
async function clickEditNode (node: DirSys) {
  if (webcontainerInstance == null) {
    return
  }
  currentEditNode.value = node
  const fileContent = await webcontainerInstance.fs.readFile(node.path, 'utf-8')
  textareaRef.value!.value = fileContent
}

const textareaRef = ref<HTMLTextAreaElement | null>(null)
const iframeRef = ref<HTMLIFrameElement | null>(null)
const terminalRef = ref<HTMLDivElement | null>(null)

function handleTextAreaInput(e: Event) {
  const textarea = e.currentTarget as HTMLTextAreaElement
  writeFile(textarea.value)
}

onMounted(async () => {
  if (textareaRef.value === null) {
    throw new Error('textareaRef is null')
  }
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
    refreshDir()
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
  // const file = await webcontainerInstance.fs.readFile('/index.js')
  // 寫入檔案
  // await webcontainerInstance.fs.writeFile('/index.js', "console.log('Hello World')")
  // recursive參數為true，可以支援建立多層資料夾
  // await webcontainerInstance.fs.mkdir('/testFolder/childFolder', { recursive: true })
  // await webcontainerInstance.fs.writeFile('/testFolder/childFolder/index.js', "console.log('Hello World')") 
  // 讀取檔案夾 回傳一個 DirEnt[]
  // const dirEnds = await webcontainerInstance.fs.readdir('/', { withFileTypes: true })
  // recursive dir return all dir
  
  // npm i
  // const exitCode = await installDependencies(terminal)
  // if (exitCode !== 0) {
  //   throw new Error('Installation failed')
  // }

  // pnpm run start
  // startDevServer(terminal) 
  refreshDir()
  
  startShell(terminal)
})

type DirSys = {
  path: string
  name: string
  child: DirSys[]
}

async function recursiveSearchDir(dirPath: string): Promise<DirSys[]> {
  const arr = []
  if (webcontainerInstance == null) {
    throw new Error('webcontainerInstance is null')
  }
  const dirEnds = await webcontainerInstance.fs.readdir(dirPath, { withFileTypes: true })
  for(let dirEnd of dirEnds ) {
    const newPath = dirPath === '/' ? `${dirPath}${dirEnd.name}` : `${dirPath}/${dirEnd.name}`
    if (dirEnd.isDirectory()) {
      if (dirEnd.name === 'node_modules') continue
      const a = await recursiveSearchDir(newPath)
      arr.push({ path: newPath, name: dirEnd.name, child: a })
    } else {
      arr.push({ path: newPath, name: dirEnd.name, child: [] })
    }
  }
  return arr
}

async function refreshDir() {
  currentDirs.value = await recursiveSearchDir('/')
}

async function startShell(terminal: Terminal) {
  if (webcontainerInstance == null) {
    throw new Error('webcontainerInstance is null')
  }
  // 使用 jsh 作為自定義的 shell
  const shellProcess = await webcontainerInstance.spawn('jsh')
  // 將 shellProcess 的 output 輸出到 terminal
  shellProcess.output.pipeTo(
    new WritableStream({
      write(data) {
        terminal.write(data)
      },
    })
  );
  // 將 terminal 下的指令輸入到 shellProcess 的 input
  const input = shellProcess.input.getWriter()
  terminal.onData((data) => {
    input.write(data)
  })
  return shellProcess
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
  // webcontainerInstance.on('server-ready', (port, url) => {
  //   if (iframeRef.value == null) {
  //     throw new Error('iframeRef is null')
  //   }
  //   iframeRef.value.src = url
  // })
}

async function writeFile(content: string) {
  if (webcontainerInstance === null) {
    throw new Error('webcontainerInstance is null')
  }
  if (currentEditNode.value === null) {
    throw new Error('currentEditNode is null')
  }
  await webcontainerInstance.fs.writeFile(currentEditNode.value?.path, content)
}

</script>