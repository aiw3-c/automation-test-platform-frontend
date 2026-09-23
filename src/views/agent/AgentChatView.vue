<template>
  <div class="agent-chat" data-testid="agent-chat.view">
    <el-row :gutter="0" style="height: calc(100vh - 60px)">
      <!-- 左侧栏 -->
      <el-col :span="5" style="border-right: 1px solid #e4e7ed; height: 100%; overflow-y: auto">
        <!-- Agent选择 -->
        <el-card shadow="never" style="margin:0; border:none; border-bottom:1px solid #e4e7ed">
          <template #header><span style="font-weight:bold;font-size:13px">Agent</span></template>
          <div v-for="agent in agents" :key="agent.name" class="agent-item"
               :class="{ active: selectedAgent === agent.name }" @click="selectAgent(agent.name)">
            <div style="font-size:13px">{{ agent.name }}</div>
            <div style="font-size:11px;color:#909399">{{ agent.description }}</div>
          </div>
        </el-card>

        <!-- 最近会话（恢复历史对话） -->
        <el-card v-if="selectedAgent" shadow="never" style="margin:0; border:none; border-bottom:1px solid #e4e7ed">
          <template #header>
            <div style="display:flex;justify-content:space-between;align-items:center">
              <span style="font-weight:bold;font-size:13px">会话</span>
              <el-button size="small" type="primary" link @click="newSession">+ 新对话</el-button>
            </div>
          </template>
          <div v-if="!sessions.length" style="color:#909399;font-size:12px;padding:4px 0">暂无历史会话</div>
          <div v-for="s in sessions" :key="s.id" class="agent-item"
               :class="{ active: currentSessionId === s.id }" @click="selectSession(s)"
               style="display:flex;justify-content:space-between;align-items:center">
            <div style="flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font-size:12px">
              {{ s.title || s.id.substring(0, 8) }}
            </div>
            <div style="font-size:10px;color:#c0c4cc;margin-left:4px">
              {{ s.updatedAt ? new Date(s.updatedAt).toLocaleDateString() : '' }}
            </div>
          </div>
        </el-card>

        <!-- Skill选择 -->
        <el-card shadow="never" style="margin:0; border:none; border-bottom:1px solid #e4e7ed">
          <template #header><span style="font-weight:bold;font-size:13px">Skills</span></template>
          <el-checkbox-group v-model="selectedSkills">
            <div v-for="skill in skills" :key="skill.name" style="margin-bottom:4px">
              <el-checkbox :label="skill.name" style="font-size:12px">{{ skill.description }}</el-checkbox>
            </div>
          </el-checkbox-group>
        </el-card>

        <!-- MCP工具状态 -->
        <el-card shadow="never" style="margin:0; border:none">
          <template #header><span style="font-weight:bold;font-size:13px">MCP Tools</span></template>
          <div v-for="tool in mcpTools" :key="tool.name" class="tool-item">
            <el-tag :type="tool.available ? 'success' : 'danger'" size="small">
              {{ tool.available ? '●' : '○' }}
            </el-tag>
            <span style="margin-left:6px;font-size:12px">{{ tool.name }}</span>
          </div>
          <div v-if="!mcpTools.length" style="color:#909399;font-size:12px">暂无工具</div>
        </el-card>
      </el-col>

      <!-- 主聊天区 -->
      <el-col :span="19" style="display:flex;flex-direction:column;height:100%">
        <!-- 状态指示器 -->
        <div class="sse-status-bar" v-if="sseStatus !== 'idle'">
          <div class="status-indicator" :class="sseStatus">
            <span v-if="sseStatus === 'thinking'" class="thinking-dot"></span>
            <span v-else-if="sseStatus === 'completed'" class="completed-icon">✓</span>
            <span class="status-text">
              {{ sseStatus === 'thinking' ? '正在思考中...' : '任务完成' }}
            </span>
          </div>
        </div>

        <!-- 消息列表 -->
        <div class="message-list" ref="messageListRef" style="flex:1;overflow-y:auto;padding:16px">
          <div v-if="!messages.length" style="text-align:center;padding:60px 0;color:#909399">
            <div style="font-size:48px;margin-bottom:16px">🤖</div>
            <div>选择Agent并开始对话</div>
          </div>
          <div v-for="(msg, idx) in messages" :key="idx" class="message-row" :class="msg.role">
            <div class="message-bubble" :class="msg.role">
              <div v-if="msg.role === 'thinking'" class="thinking-bubble">
                <div class="thinking-header" @click="toggleThinking(msg)">
                  🤔 {{ msg.content.substring(0, 50) }}{{ msg.content.length > 50 ? '...' : '' }}
                  <span class="thinking-toggle">{{ msg.expanded ? '▼' : '▶' }}</span>
                </div>
                <div v-if="msg.expanded" class="thinking-content">
                  {{ msg.content }}
                </div>
              </div>
              <div v-else-if="msg.role === 'reasoning'" class="reasoning-bubble">
                <div class="reasoning-header" @click="toggleThinking(msg)">
                  💡 思考过程 {{ msg.content.length > 50 ? '...' : '' }}
                  <span class="thinking-toggle">{{ msg.expanded ? '▼' : '▶' }}</span>
                </div>
                <div v-if="msg.expanded" class="reasoning-content">
                  {{ msg.content }}
                </div>
              </div>
              <div v-else-if="msg.role === 'tool_call'" class="tool-call-bubble">
                <div class="tool-call-header">
                  🔧 调用工具: <code>{{ msg.toolName }}</code>
                  <span v-if="msg.status === 'executing'" class="tool-status executing">执行中...</span>
                  <span v-else-if="msg.status === 'completed'" class="tool-status completed">✓ 执行完成</span>
                </div>
                <div v-if="msg.toolResult" class="tool-result-inline">
                  <div v-if="msg.error" class="tool-error">❌ {{ msg.error }}</div>
                  <div v-else-if="isSqlResult(msg.toolResult)" class="sql-result">
                    <pre>{{ extractSqlText(msg.toolResult) }}</pre>
                  </div>
                  <div v-else-if="isSiyuanNotebooks(msg.toolName, msg.toolResult)" class="siyuan-notebooks">
                    <div v-for="(nb, idx) in parseSiyuanNotebooks(msg.toolResult)" :key="idx" class="notebook-item">
                      <span class="notebook-icon">📒</span>
                      <span class="notebook-name">{{ nb.name }}</span>
                      <span class="notebook-id">{{ nb.id }}</span>
                    </div>
                  </div>
                  <div v-else-if="isSiyuanSearch(msg.toolName, msg.toolResult)" class="siyuan-search">
                    <div class="search-summary">共找到 {{ parseSiyuanSearch(msg.toolResult).total }} 条结果</div>
                    <div v-for="(item, idx) in parseSiyuanSearch(msg.toolResult).items" :key="idx" class="search-item">
                      <div class="search-title">{{ item.title || item.docId }}</div>
                      <div class="search-snippet">{{ item.snippet }}</div>
                      <div class="search-meta">
                        <span>{{ item.hPath }}</span>
                        <span v-if="item.score">分数: {{ item.score }}</span>
                      </div>
                    </div>
                  </div>
                  <pre v-else class="tool-result-json">{{ formatToolResult(msg.toolResult) }}</pre>
                </div>
              </div>
              <div v-else v-html="renderMarkdown(msg.content)"></div>
            </div>
            <div class="message-role">{{ msg.role === 'user' ? '你' : 'AI' }}</div>
          </div>
          <div v-if="isStreaming" class="message-row assistant">
            <div class="message-bubble assistant streaming-cursor" v-html="renderMarkdown(streamingText)"></div>
            <div class="message-role">AI</div>
          </div>
        </div>

        <!-- 输入区 -->
        <div class="input-area" style="padding:12px 16px;border-top:1px solid #e4e7ed">
          <!-- @项目选择弹窗 -->
          <div v-if="showProjectPopup" class="popup-menu" data-testid="agent-chat.view.popup-project">
            <div class="popup-header">选择项目</div>
            <div v-for="p in projects" :key="p.id" class="popup-item" @click="insertProject(p)">
              {{ p.name }}
            </div>
          </div>
          <!-- /Skill选择弹窗 -->
          <div v-if="showSkillPopup" class="popup-menu" data-testid="agent-chat.view.popup-skill">
            <div class="popup-header">选择Skill</div>
            <div v-for="s in allSkills" :key="s.name" class="popup-item" @click="insertSkill(s)">
              {{ s.name }} - {{ s.description }}
            </div>
          </div>
          <div style="display:flex;gap:8px">
            <el-input v-model="inputMessage" type="textarea" :rows="2"
                      placeholder="输入消息... @选项目 /选Skill Enter发送"
                      @keydown.enter.exact.prevent="sendMessage"
                      @input="handleInput"
                      :disabled="!selectedAgent"
                      ref="chatInputRef"
                      data-testid="agent-chat.view.input" />
            <el-button type="primary" @click="sendMessage"
                       :loading="isStreaming" :disabled="!selectedAgent || !inputMessage.trim()"
                       data-testid="agent-chat.view.btn-send">
              发送
            </el-button>
          </div>
        </div>
      </el-col>
    </el-row>

    <!-- 安全拦截确认弹窗 -->
    <el-dialog v-model="safetyDialogVisible" :title="safetyCheckInfo?.blocked ? '🚫 操作已阻止' : '⚠️ 操作确认'" width="550px" :close-on-click-modal="false" data-testid="agent-chat.view.dialog-safety">
      <div v-if="safetyCheckInfo">
        <el-alert v-if="safetyCheckInfo.blocked" type="error" :closable="false" style="margin-bottom:16px">
          <template #title>该操作已被安全策略阻止</template>
          <div>{{ safetyCheckInfo.message }}</div>
          <div v-if="safetyCheckInfo.riskDetail" style="margin-top:8px;color:#909399;font-size:13px">{{ safetyCheckInfo.riskDetail }}</div>
        </el-alert>
        <el-alert v-else type="warning" :closable="false" style="margin-bottom:16px">
          <template #title>{{ safetyCheckInfo.message }}</template>
        </el-alert>

        <div style="margin-bottom:12px">
          <el-tag :type="safetyCheckInfo.level === 'DANGEROUS' ? 'danger' : 'warning'" size="small">
            {{ safetyCheckInfo.level }}
          </el-tag>
          <span style="margin-left:8px;color:#909399;font-size:12px">规则: {{ safetyCheckInfo.ruleName }} | 关键词: {{ safetyCheckInfo.keyword }}</span>
        </div>

        <div v-if="modifyInputVisible" style="margin-bottom:16px">
          <el-input v-model="modifyMessage" type="textarea" :rows="3" placeholder="请输入修改后的内容" data-testid="agent-chat.view.input-modify" />
        </div>
      </div>
      <template #footer>
        <div v-if="safetyCheckInfo?.blocked">
          <el-button type="info" @click="handleSafetyAction('acknowledge')">了解</el-button>
        </div>
        <div v-else>
          <el-button v-if="modifyInputVisible" type="warning" @click="handleSafetyAction('modify')">确认修改执行</el-button>
          <el-button v-if="!modifyInputVisible" type="warning" @click="modifyInputVisible = true">修改后执行</el-button>
          <el-button type="primary" @click="handleSafetyAction('confirm')">确认执行</el-button>
          <el-button type="danger" @click="handleSafetyAction('cancel')">取消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from 'vue'
import { getAgentList, getAgentSkillList, getAgentSessions, getChatMessages } from '@/api/agent'
import request from '@/utils/request'
import Cookies from 'js-cookie'
import { marked } from 'marked'

const agents = ref([])
const skills = ref([])
const mcpTools = ref([])
const selectedAgent = ref('')
const selectedSkills = ref([])
const messages = ref([])
const inputMessage = ref('')
const isStreaming = ref(false)
const streamingText = ref('')
const messageListRef = ref(null)
const currentSessionId = ref('')
const safetyDialogVisible = ref(false)
const safetyCheckInfo = ref(null)
const modifyInputVisible = ref(false)
const modifyMessage = ref('')
const showProjectPopup = ref(false)
const showSkillPopup = ref(false)
const projects = ref([])
const allSkills = ref([])
const chatInputRef = ref(null)
const sseStatus = ref('idle')
const taskCompletedTimer = ref(null)
const sessions = ref([])

function handleInput() {
  const val = inputMessage.value
  if (val.endsWith('@')) {
    showProjectPopup.value = true
    showSkillPopup.value = false
  } else if (val.endsWith('/')) {
    showSkillPopup.value = true
    showProjectPopup.value = false
  } else {
    showProjectPopup.value = false
    showSkillPopup.value = false
  }
}

function insertProject(project) {
  inputMessage.value = inputMessage.value.replace(/@$/, '') + `@${project.name} `
  showProjectPopup.value = false
}

function insertSkill(skill) {
  inputMessage.value = inputMessage.value.replace(/\/$/, '') + `/${skill.name} `
  showSkillPopup.value = false
}

async function loadProjects() {
  try {
    const res = await request.get('/projects', { params: { page: 0, size: 50 } })
    projects.value = res.data?.content || []
  } catch { projects.value = [] }
}

async function loadAllSkills() {
  try {
    const res = await request.get('/agents/skills')
    allSkills.value = res.data || []
  } catch { allSkills.value = [] }
}

async function loadAgents() {
  const res = await getAgentList()
  agents.value = res.data || []
}

async function loadSkills() {
  try {
    const res = await request.get('/agents/skills')
    skills.value = res.data || []
  } catch { skills.value = [] }
}

async function loadMcpTools() {
  try {
    const res = await request.get('/agents/mcp/tools')
    mcpTools.value = (res.data || []).map(t => ({
      name: t.name,
      description: t.description,
      serverId: t.serverId,
      available: true
    }))
  } catch {
    // 降级到静态列表
    mcpTools.value = [
      { name: 'read-swagger-spec', description: '读取Swagger文档', serverId: 'swagger', available: true },
      { name: 'list_all_datasources', description: '列出数据源', serverId: 'chat2db', available: true },
      { name: 'execute_sql', description: '执行SQL查询', serverId: 'chat2db', available: true },
      { name: 'text2sql', description: '自然语言转SQL', serverId: 'chat2db', available: true },
    ]
  }
}

function selectAgent(name) {
  selectedAgent.value = name
  messages.value = []
  streamingText.value = ''
  currentSessionId.value = crypto.randomUUID()
  // 切换Agent后加载该Agent的最近会话列表
  loadSessions(name)
}

async function loadSessions(name) {
  const agentName = name || selectedAgent.value
  if (!agentName) return
  try {
    const res = await getAgentSessions(agentName)
    sessions.value = res.data || []
    // 自动选中最近会话（如有），并加载其消息；否则保持新会话状态
    if (sessions.value.length > 0) {
      await selectSession(sessions.value[0])
    }
  } catch {
    sessions.value = []
  }
}

async function selectSession(session) {
  if (!session || !session.id) return
  currentSessionId.value = session.id
  messages.value = []
  streamingText.value = ''
  if (!selectedAgent.value) return
  try {
    const res = await getChatMessages(selectedAgent.value, session.id)
    const list = res.data || []
    // 将后端 ChatMessage 列表映射为前端消息结构
    messages.value = list
      .filter(m => m.role === 'user' || m.role === 'assistant')
      .map(m => ({ role: m.role, content: m.content }))
  } catch {
    // 加载历史失败时保持空消息列表
  }
  await nextTick()
  scrollToBottom()
}

function newSession() {
  messages.value = []
  streamingText.value = ''
  currentSessionId.value = crypto.randomUUID()
}

function renderMarkdown(text) {
  if (!text) return ''
  try {
    marked.setOptions({
      breaks: true,
      gfm: true,
      headerIds: false,
      mangle: false
    })
    let parsed = marked.parse(text)
    parsed = parsed.replace(/<p>\s*<\/p>/g, '<p>&nbsp;</p>')
    return parsed
  } catch (e) {
    console.error('Markdown渲染错误:', e)
    return text.replace(/\n/g, '<br>')
  }
}

function isSqlResult(result) {
  if (!result) return false
  const text = typeof result === 'string' ? result : JSON.stringify(result)
  return text.includes('sqlType') || text.includes('rows:') || text.includes('durationMs')
}

function extractSqlText(result) {
  if (!result) return ''
  if (typeof result === 'string') {
    try {
      const parsed = JSON.parse(result)
      if (parsed.result && parsed.result.content) {
        return parsed.result.content.map(c => c.text || '').join('\n')
      }
    } catch {}
    return result
  }
  return JSON.stringify(result, null, 2)
}

async function sendMessage() {
  if (!inputMessage.value.trim() || isStreaming.value || !selectedAgent.value) return

  const msg = inputMessage.value.trim()
  messages.value.push({ role: 'user', content: msg })
  inputMessage.value = ''
  isStreaming.value = true
  streamingText.value = ''
  sseStatus.value = 'thinking'
  if (taskCompletedTimer.value) {
    clearTimeout(taskCompletedTimer.value)
    taskCompletedTimer.value = null
  }
  await nextTick()
  scrollToBottom()

  const body = JSON.stringify({
    sessionId: currentSessionId.value,
    message: msg,
    selectedSkills: selectedSkills.value
  })
  const headers = { 'Content-Type': 'application/json', 'Authorization': `Bearer ${Cookies.get('token') || ''}` }

  // 先尝试SSE流式
  try {
    const response = await fetch(`/api/v1/agents/${selectedAgent.value}/chat/stream`, {
      method: 'POST', headers, body
    })

    if (response.ok && response.headers.get('Content-Type')?.includes('text/event-stream')) {
      await handleSSE(response)
      return
    }
  } catch (e) {
    // SSE失败，fallback到同步模式
  }

  // Fallback: 同步模式
  try {
    const response = await fetch(`/api/v1/agents/${selectedAgent.value}/chat`, {
      method: 'POST', headers, body
    })
    const data = await response.json()
    if (data.code === 200 && data.data) {
      const result = data.data
      if (result.safetyCheck && result.safetyCheck.intercepted) {
        // 安全拦截 — 显示拦截消息 + 确认弹窗
        messages.value.push({ role: 'assistant', content: result.reply })
        showSafetyDialog(result.safetyCheck)
      } else {
        messages.value.push({ role: 'assistant', content: result.reply })
      }
    } else {
      messages.value.push({ role: 'assistant', content: `❌ ${data.message || '请求失败'}` })
    }
  } catch (e) {
    messages.value.push({ role: 'assistant', content: `❌ 请求失败: ${e.message}` })
  } finally {
    isStreaming.value = false
  }
}

async function handleSSE(response) {
  const reader = response.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''
  let currentEvent = ''

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split('\n')
    buffer = lines.pop() || ''

    for (const line of lines) {
      const trimmed = line.trim()
      if (trimmed.startsWith('event:')) {
        currentEvent = trimmed.substring(6).trim()
      } else if (trimmed.startsWith('data:')) {
        const dataStr = trimmed.substring(5).trim()
        handleSSEEvent(currentEvent, dataStr)
        currentEvent = ''
      }
    }
  }
  isStreaming.value = false
  sseStatus.value = 'completed'
  if (taskCompletedTimer.value) {
    clearTimeout(taskCompletedTimer.value)
  }
  taskCompletedTimer.value = setTimeout(() => {
    sseStatus.value = 'idle'
    taskCompletedTimer.value = null
  }, 2000)
}

function toggleThinking(msg) {
  msg.expanded = !msg.expanded
}

function handleSSEEvent(event, data) {
  switch (event) {
    case 'thinking':
      sseStatus.value = 'thinking'
      messages.value.push({ role: 'thinking', content: data, expanded: false })
      streamingText.value = ''
      break
    case 'reasoning':
      messages.value.push({ role: 'reasoning', content: data, expanded: false })
      streamingText.value = ''
      break
    case 'token':
      try {
        const d = JSON.parse(data)
        streamingText.value += (d.content || d) + '\n\n'
      } catch {
        streamingText.value += data + '\n\n'
      }
      break
    case 'tool':
      try {
        const d = JSON.parse(data)
        messages.value.push({ 
          role: 'tool_call', 
          toolName: d.tool, 
          status: d.status || 'executing', 
          content: '',
          toolResult: null,
          error: null
        })
      } catch {}
      break
    case 'tool_result':
      try {
        const d = JSON.parse(data)
        const lastToolCall = [...messages.value].reverse().find(m => m.role === 'tool_call' && m.toolName === d.tool && !m.toolResult)
        if (lastToolCall) {
          lastToolCall.toolResult = d.result
          lastToolCall.error = d.error
          lastToolCall.status = 'completed'
        } else {
          messages.value.push({ 
            role: 'tool_call', 
            toolName: d.tool, 
            status: 'completed', 
            content: '',
            toolResult: d.result,
            error: d.error
          })
        }
      } catch {
        const lastToolCall = [...messages.value].reverse().find(m => m.role === 'tool_call' && !m.toolResult)
        if (lastToolCall) {
          lastToolCall.toolResult = data
          lastToolCall.status = 'completed'
        } else {
          messages.value.push({ 
            role: 'tool_call', 
            toolName: 'unknown', 
            status: 'completed', 
            content: '',
            toolResult: data,
            error: null
          })
        }
      }
      break
    case 'done':
      try {
        const d = JSON.parse(data)
        if (streamingText.value) {
          messages.value.push({ role: 'assistant', content: streamingText.value })
        } else if (d.fullResponse) {
          messages.value.push({ role: 'assistant', content: d.fullResponse })
        }
        streamingText.value = ''
      } catch {
        if (streamingText.value) {
          messages.value.push({ role: 'assistant', content: streamingText.value })
        }
        streamingText.value = ''
      }
      break
    case 'error':
      try {
        const d = JSON.parse(data)
        const lastToolCall = [...messages.value].reverse().find(m => m.role === 'tool_call' && m.status === 'executing')
        if (lastToolCall) {
          lastToolCall.error = d.error
          lastToolCall.status = 'completed'
        } else {
          messages.value.push({ role: 'assistant', content: `❌ ${d.error}` })
        }
      } catch {
        messages.value.push({ role: 'assistant', content: '❌ 工具执行失败' })
      }
      streamingText.value = ''
      break
  }
  nextTick(() => scrollToBottom())
}

function showSafetyDialog(info) {
  safetyCheckInfo.value = info
  safetyDialogVisible.value = true
  modifyInputVisible.value = false
  modifyMessage.value = ''
}

function isSiyuanNotebooks(toolName, result) {
  if (!toolName || !result) return false
  return toolName.startsWith('siyuan_list_notebooks') || 
         (toolName.startsWith('siyuan') && typeof result === 'string')
}

function parseSiyuanNotebooks(result) {
  try {
    const data = typeof result === 'string' ? JSON.parse(result) : result
    if (Array.isArray(data)) return data
    if (data.data && Array.isArray(data.data)) return data.data
    return []
  } catch {
    return []
  }
}

function isSiyuanSearch(toolName, result) {
  if (!toolName || !result) return false
  return toolName.startsWith('siyuan_search') || 
         (toolName.startsWith('siyuan') && typeof result === 'object' && result.items)
}

function parseSiyuanSearch(result) {
  try {
    const data = typeof result === 'string' ? JSON.parse(result) : result
    if (data.items) {
      return {
        total: data.total || 0,
        items: Array.isArray(data.items) ? data.items : []
      }
    }
    return { total: 0, items: [] }
  } catch {
    return { total: 0, items: [] }
  }
}

function formatToolResult(result) {
  if (typeof result === 'string') {
    try {
      const obj = JSON.parse(result)
      return JSON.stringify(obj, null, 2)
    } catch {
      return result
    }
  }
  return JSON.stringify(result, null, 2)
}

async function handleSafetyAction(action) {
  safetyDialogVisible.value = false
  if (action === 'acknowledge') return

  const body = {
    sessionId: currentSessionId.value,
    action: action,
    modifiedMessage: action === 'modify' ? modifyMessage.value : undefined
  }
  const headers = { 'Content-Type': 'application/json', 'Authorization': `Bearer ${Cookies.get('token') || ''}` }

  isStreaming.value = true
  try {
    const response = await fetch(`/api/v1/agents/${selectedAgent.value}/chat/confirm`, {
      method: 'POST', headers,
      body: JSON.stringify(body)
    })
    const data = await response.json()
    if (data.code === 200 && data.data) {
      if (data.data.safetyCheck && data.data.safetyCheck.intercepted) {
        messages.value.push({ role: 'assistant', content: data.data.reply })
        showSafetyDialog(data.data.safetyCheck)
      } else {
        messages.value.push({ role: 'assistant', content: data.data.reply })
      }
    }
  } catch (e) {
    messages.value.push({ role: 'assistant', content: `❌ 确认请求失败: ${e.message}` })
  } finally {
    isStreaming.value = false
  }
}

function scrollToBottom() {
  if (messageListRef.value) {
    messageListRef.value.scrollTop = messageListRef.value.scrollHeight
  }
}

onMounted(async () => {
  await loadAgents()
  loadSkills()
  loadMcpTools()
  loadProjects()
  loadAllSkills()
  // 默认加载第一个Agent的最近会话，恢复历史对话
  if (agents.value.length > 0) {
    selectedAgent.value = agents.value[0].name
    currentSessionId.value = crypto.randomUUID()
    await loadSessions()
  }
})
</script>

<style scoped>
.agent-chat { height: 100%; }
.agent-item { padding: 8px; cursor: pointer; border-radius: 4px; margin-bottom: 4px; }
.agent-item:hover { background: #f5f7fa; }
.agent-item.active { background: #ecf5ff; color: #409eff; }
.tool-item { display: flex; align-items: center; padding: 4px 0; }
.message-row { display: flex; gap: 8px; margin-bottom: 16px; flex-direction: column; }
.message-row.user { align-items: flex-end; }
.message-row.assistant, .message-row.tool_call, .message-row.tool_result, .message-row.thinking { align-items: flex-start; }
.message-bubble { max-width: 75%; padding: 12px 16px; border-radius: 12px; font-size: 14px; line-height: 1.6; word-break: break-word; }
.message-bubble.user { background: #409eff; color: white; border-bottom-right-radius: 4px; }
.message-bubble.assistant { background: #f4f4f5; color: #303133; border-bottom-left-radius: 4px; }
.message-bubble.assistant code { background: #e4e7ed; padding: 2px 4px; border-radius: 3px; font-size: 13px; }
.message-bubble.assistant pre { background: #1e1e1e; color: #d4d4d4; padding: 12px; border-radius: 6px; overflow-x: auto; margin: 8px 0; font-size: 13px; }
.thinking-bubble { background: #ecf5ff; color: #409eff; border: 1px solid #dbeafe; }
.thinking-header { cursor: pointer; display: flex; justify-content: space-between; align-items: center; }
.thinking-toggle { font-size: 12px; margin-left: 8px; transition: transform 0.2s; }
.thinking-content { margin-top: 8px; padding-top: 8px; border-top: 1px dashed #dbeafe; font-size: 13px; line-height: 1.6; }
.reasoning-bubble { background: #fdf6ec; color: #e6a23c; border: 1px solid #faecd8; }
.reasoning-header { cursor: pointer; display: flex; justify-content: space-between; align-items: center; }
.reasoning-content { margin-top: 8px; padding-top: 8px; border-top: 1px dashed #faecd8; font-size: 13px; line-height: 1.6; }
.tool-call-bubble { background: #fdf6ec; color: #e6a23c; border: 1px solid #faecd8; }
.tool-call-bubble code { background: #faecd8; padding: 2px 4px; border-radius: 3px; }
.tool-call-header { display: flex; align-items: center; flex-wrap: wrap; }
.tool-status { margin-left: 8px; font-size: 12px; color: #909399; }
.tool-status.executing { color: #e6a23c; }
.tool-status.completed { color: #67c23a; }
.tool-result-inline { margin-top: 10px; padding-top: 10px; border-top: 1px dashed #faecd8; }
.tool-result-inline pre { background: #f5f7fa; padding: 8px; border-radius: 4px; margin: 4px 0; font-size: 12px; max-height: 200px; overflow-y: auto; }
.tool-result-bubble { background: #f0f9eb; color: #67c23a; border: 1px solid #e1f3d8; }
.tool-result-bubble pre { background: #f5f7fa; padding: 8px; border-radius: 4px; margin: 4px 0; font-size: 12px; max-height: 200px; overflow-y: auto; }

.sse-status-bar {
  padding: 8px 16px;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  align-items: center;
}

.status-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  font-weight: 500;
}

.status-indicator.thinking { color: #409eff; }
.status-indicator.completed { color: #67c23a; }

.thinking-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  background: #409eff;
  border-radius: 50%;
  animation: pulse 1.5s infinite;
}

.completed-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  background: #67c23a;
  color: white;
  border-radius: 50%;
  font-size: 12px;
  font-weight: bold;
}

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(1.2); }
}
.tool-error { color: #f56c6c; font-size: 13px; margin-top: 4px; padding: 4px 8px; background: #fef0f0; border-radius: 4px; }
.sql-result pre { background: #1e1e1e; color: #d4d4d4; padding: 12px; border-radius: 6px; font-size: 12px; white-space: pre-wrap; }
.tool-result-json { background: #f5f7fa; padding: 8px; border-radius: 4px; margin: 4px 0; font-size: 12px; max-height: 200px; overflow-y: auto; }
.siyuan-notebooks { margin-top: 8px; }
.notebook-item { display: flex; align-items: center; gap: 8px; padding: 6px 0; border-bottom: 1px dashed #e1f3d8; }
.notebook-item:last-child { border-bottom: none; }
.notebook-icon { font-size: 16px; }
.notebook-name { flex: 1; font-weight: 500; }
.notebook-id { font-size: 11px; color: #909399; font-family: monospace; }
.siyuan-search { margin-top: 8px; }
.search-summary { font-size: 12px; color: #909399; margin-bottom: 8px; }
.search-item { background: #f5f7fa; padding: 8px 12px; border-radius: 6px; margin-bottom: 8px; }
.search-item:last-child { margin-bottom: 0; }
.search-title { font-weight: 500; margin-bottom: 4px; }
.search-snippet { font-size: 13px; color: #606266; margin-bottom: 4px; }
.search-meta { font-size: 11px; color: #909399; display: flex; gap: 12px; }
.message-role { font-size: 11px; color: #909399; margin-top: 2px; }
.streaming-cursor::after { content: '▊'; animation: blink 1s infinite; }
@keyframes blink { 0%,100% { opacity: 1; } 50% { opacity: 0; } }
.popup-menu { position: absolute; bottom: 100%; left: 0; background: white; border: 1px solid #e4e7ed; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); max-height: 250px; overflow-y: auto; width: 350px; z-index: 100; padding: 8px 0; }
.popup-header { padding: 4px 12px; font-weight: bold; font-size: 12px; color: #909399; border-bottom: 1px solid #f0f0f0; margin-bottom: 4px; }
.popup-item { padding: 8px 12px; cursor: pointer; font-size: 13px; }
.popup-item:hover { background: #f5f7fa; }
.input-area { position: relative; }
</style>
