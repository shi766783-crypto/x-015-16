<script setup>
import { computed, ref } from 'vue'
import { Archive, ArrowLeft, Award, Bell, BookOpen, CalendarClock, ChefHat, ChevronRight, CircleAlert, ClipboardList, FilePlus2, Flame, FolderLock, ImageOff, LayoutDashboard, Menu, Plus, RotateCcw, Search, ShieldCheck, Star, TrendingUp, UserRound, X } from 'lucide-vue-next'
import { config, days } from './data'
import { useKeeper } from './composables/useKeeper'
const { state, add, update, metrics, reset } = useKeeper()
const active = ref('overview'); const query = ref(''); const category = ref(''); const selectedId = ref(null); const showForm = ref(false); const mobileOpen = ref(false); const draft = ref({})
const activeNav = computed(() => config.nav.find((x) => x.id === active.value))
const docs = computed(() => state.documents.filter((x) => `${x.name}${x.holder}${x.category}`.toLowerCase().includes(query.value.toLowerCase()) && (!category.value || x.category === category.value)))
const selectedRecipe = computed(() => state.documents.find((x) => x.id === selectedId.value) || null)
const selectedCooks = computed(() => selectedRecipe.value ? state.usages.filter((x) => x.documentId === selectedRecipe.value.id).sort((a, b) => b.date.localeCompare(a.date)) : [])
const cookStats = computed(() => {
  const list = selectedCooks.value
  const rated = list.filter((x) => Number(x.score) > 0)
  const avg = rated.length ? (rated.reduce((a, x) => a + Number(x.score), 0) / rated.length).toFixed(1) : '—'
  return { total: list.length, avg, rated: rated.length }
})
const splitItems = (text) => String(text || '').split(/[、,，;；\n]/).map((x) => x.trim()).filter(Boolean)
const alerts = computed(() => state.documents.map((x) => ({ ...x, remaining: days(x.expires) })).filter((x) => x.remaining <= 90).sort((a, b) => a.remaining - b.remaining))
const borrowed = computed(() => state.usages.filter((x) => x.status === '借用中'))
const completion = computed(() => Math.min(100, Math.round((state.documents.filter((x) => x.location && x.copyLocation).length / Math.max(state.documents.length, 1)) * 100)))
function openCreate() { draft.value = Object.fromEntries(config.form.fields.map(([key, , value]) => [key, value || ''])); showForm.value = true }
function save() { add(config.form.collection, { ...draft.value }); showForm.value = false }
function readFile(event) { const file = event.target.files?.[0]; if (!file) return; const reader = new FileReader(); reader.onload = () => { draft.value.photo = reader.result }; reader.readAsDataURL(file) }
function markRenewed(doc) { update('documents', doc.id, { expires: `${new Date().getFullYear() + 10}-12-31`, renewed: true }) }
function navTo(id) { active.value = id; mobileOpen.value = false; selectedId.value = null }
function openDetail(item) { selectedId.value = item.id }
function closeDetail() { selectedId.value = null }
</script>
<template>
  <div class="app-shell">
    <aside class="sidebar" :class="{ open: mobileOpen }"><div class="brand"><span class="brand-mark"><FolderLock :size="19" /></span><div><strong>{{ config.brand }}</strong><small>{{ config.subtitle }}</small></div><button class="icon-btn close" @click="mobileOpen = false"><X :size="17" /></button></div><div class="family-chip"><span>林</span><div><b>林家家庭空间</b><small>4 位成员 · 私密存储</small></div><ChevronRight :size="15" /></div><nav><button v-for="item in config.nav" :key="item.id" :class="{ active: active === item.id }" @click="navTo(item.id)"><component :is="item.id === 'overview' ? LayoutDashboard : item.id === 'documents' ? Archive : item.id === 'history' ? CalendarClock : item.id === 'insurance' ? ShieldCheck : Award" :size="17" /><span>{{ item.label }}</span><i v-if="item.id === 'documents' && alerts.length">{{ alerts.length }}</i></button></nav><div class="sidebar-foot"><button><BookOpen :size="17" />使用指南</button><button><UserRound :size="17" />个人中心</button><button class="reset" @click="reset"><RotateCcw :size="15" />恢复演示数据</button></div></aside>
    <main class="main"><header class="topbar"><button class="icon-btn menu" @click="mobileOpen = true"><Menu :size="20" /></button><div><span class="eyebrow">家庭文件空间</span><h1>{{ activeNav.label }}</h1></div><div class="top-actions"><button class="icon-btn"><Bell :size="18" /><em v-if="alerts.length"></em></button><span class="avatar">林</span><b>Lin</b></div></header>
      <div class="page">
        <section v-if="!selectedRecipe" class="intro"><div><span class="eyebrow">{{ active === 'overview' ? '星期三 · 2026年9月16日' : '数据随时保存在当前设备' }}</span><h2>{{ active === 'overview' ? config.heroTitle : activeNav.label }}</h2><p>{{ active === 'overview' ? config.heroCopy : '集中查看、编辑和追踪家庭资料。' }}</p></div><button class="primary" @click="openCreate"><Plus :size="16" />{{ config.createLabel || '新建档案' }}</button></section>
        <section v-if="selectedRecipe" class="panel detail">
          <button class="text-btn back-btn" @click="closeDetail"><ArrowLeft :size="14" />返回菜谱列表</button>
          <div class="detail-hero">
            <div class="detail-photo" :class="{ placeholder: !selectedRecipe.photo }">
              <img v-if="selectedRecipe.photo" :src="selectedRecipe.photo" :alt="selectedRecipe.name" />
              <template v-else><ImageOff :size="20" /><small>暂无照片</small></template>
            </div>
            <div class="detail-heading">
              <div class="detail-tags"><span class="tag">{{ selectedRecipe.category || '未分类' }}</span><span class="tag soft"><ChefHat :size="12" />{{ selectedRecipe.holder || '难度未填' }}</span></div>
              <h2>{{ selectedRecipe.name }}</h2>
              <div class="detail-meta">
                <span><b>{{ selectedRecipe.number || '未填写' }}</b><small>准备 / 烹饪时间</small></span>
                <span><b>{{ selectedRecipe.issued || '—' }}</b><small>最近烹饪</small></span>
                <span><b :class="{ dangerText: days(selectedRecipe.expires) < 0, warnText: days(selectedRecipe.expires) >= 0 && days(selectedRecipe.expires) <= 90 }">{{ selectedRecipe.expires || '—' }}</b><small>{{ selectedRecipe.expires ? days(selectedRecipe.expires) < 0 ? `已超过推荐日期 ${Math.abs(days(selectedRecipe.expires))} 天` : `${config.remainingPrefix || '剩余'} ${days(selectedRecipe.expires)} 天` : '下次推荐日期未填' }}</small></span>
              </div>
              <div class="detail-stats">
                <span class="stat-pill"><CalendarClock :size="14" />做过 <b>{{ cookStats.total }}</b> 次</span>
                <span class="stat-pill"><Star :size="14" class="filled" />平均 <b>{{ cookStats.avg }}</b><small v-if="cookStats.rated"> / {{ cookStats.rated }} 次评分</small></span>
              </div>
            </div>
          </div>
          <div class="detail-cols">
            <article class="detail-card">
              <div class="detail-card-head"><span class="detail-card-icon green"><ClipboardList :size="15" /></span><div><h3>食材清单</h3><small>备菜时照着准备即可</small></div></div>
              <ul v-if="splitItems(selectedRecipe.location).length" class="ingredient-list"><li v-for="ing in splitItems(selectedRecipe.location)" :key="ing">{{ ing }}</li></ul>
              <div v-else class="empty">还没有记录食材。</div>
            </article>
            <article class="detail-card">
              <div class="detail-card-head"><span class="detail-card-icon peach"><Flame :size="15" /></span><div><h3>调料清单</h3><small>用量可按口味调整</small></div></div>
              <ul v-if="splitItems(selectedRecipe.copyLocation).length" class="ingredient-list"><li v-for="ing in splitItems(selectedRecipe.copyLocation)" :key="ing">{{ ing }}</li></ul>
              <div v-else class="empty">还没有记录调料。</div>
            </article>
          </div>
          <article class="detail-card">
            <div class="detail-card-head"><span class="detail-card-icon blue"><ShieldCheck :size="15" /></span><div><h3>小贴士</h3><small>家传经验，越做越顺手</small></div></div>
            <p class="tips-text">{{ selectedRecipe.notes || '还没有小贴士，做过一次后可以补充心得。' }}</p>
          </article>
          <article class="detail-card">
            <div class="detail-card-head"><span class="detail-card-icon green"><CalendarClock :size="15" /></span><div><h3>烹饪记录</h3><small>这道菜谱名下的每次实践</small></div><span class="count-note">共 {{ cookStats.total }} 次</span></div>
            <div v-if="selectedCooks.length" class="cook-list">
              <div v-for="entry in selectedCooks" :key="entry.id" class="cook-row">
                <span class="cook-date">{{ entry.date }}<em :class="entry.status === '完成' ? 'goodText' : 'warnText'">{{ entry.status || '已记录' }}</em></span>
                <div class="cook-main"><b>{{ entry.purpose || '家常一餐' }}</b><small>经办人：{{ entry.borrower || '未填写' }}</small></div>
                <span class="cook-score"><template v-if="Number(entry.score) > 0"><Star v-for="n in 5" :key="n" :size="13" :class="{ filled: n <= Number(entry.score) }" /></template><small v-else>未评分</small></span>
                <p class="cook-note">{{ entry.note || '本次没有备注。' }}</p>
              </div>
            </div>
            <div v-else class="empty cook-empty"><ChefHat :size="20" /><p>这道菜还没有烹饪记录，做过一次后会在这里看到时间、评分和心得。</p></div>
          </article>
        </section>
        <section v-if="active === 'overview'" class="metric-grid"><article v-for="metric in metrics" :key="metric.label" class="metric"><span class="metric-icon" :class="metric.tone"><Archive v-if="metric.tone === 'green'" :size="17" /><CalendarClock v-else-if="metric.tone === 'peach'" :size="17" /><CircleAlert v-else-if="metric.tone === 'red'" :size="17" /><ShieldCheck v-else :size="17" /></span><div><small>{{ metric.label }}</small><strong>{{ metric.value }}</strong><em>{{ metric.tone === 'red' && metric.value ? '需要尽快处理' : metric.tone === 'peach' && metric.value ? '请提前安排续办' : '较上月稳定' }}</em></div></article></section>
        <section v-if="active === 'overview'" class="content-grid"><article class="panel"><div class="panel-head"><div><span class="eyebrow">到期预警</span><h3>需要你留意的文件</h3></div><button class="text-btn" @click="navTo('documents')">查看全部 <ChevronRight :size="14" /></button></div><div class="alert-list"><div v-for="item in alerts.slice(0, 4)" :key="item.id" class="alert-row"><span class="alert-icon" :class="item.remaining < 0 ? 'danger' : 'warn'"><CircleAlert :size="16" /></span><div><b>{{ item.name }}</b><small>{{ item.holder }} · {{ item.location }}</small></div><span class="date-pill" :class="item.remaining < 0 ? 'danger' : 'warn'">{{ item.remaining < 0 ? `已过期 ${Math.abs(item.remaining)} 天` : `${item.remaining} 天后到期` }}</span><button v-if="item.remaining <= 90" class="tiny-btn" @click="markRenewed(item)">已续办</button></div><div v-if="!alerts.length" class="empty">目前没有临期文件，保持得很好。</div></div></article><article class="panel score"><div class="panel-head"><div><span class="eyebrow">档案完整度</span><h3>家庭资料健康度</h3></div><TrendingUp :size="18" class="accent" /></div><div class="score-ring"><strong>{{ completion }}<small>%</small></strong></div><p>每一份文件都有存放位置和备份位置，家人需要时可以快速找到。</p><div class="progress"><span :style="{ width: `${completion}%` }"></span></div><small class="muted">已完善 {{ state.documents.filter(x => x.location && x.copyLocation).length }} / {{ state.documents.length }} 份</small></article></section>
        <section v-else-if="active === 'documents' && !selectedRecipe" class="panel"><div class="toolbar"><div class="search"><Search :size="16" /><input v-model="query" :placeholder="`搜索${config.entityLabel}`" /></div><select v-model="category"><option value="">全部类别</option><option v-for="item in config.categories" :key="item" :value="item">{{ item }}</option></select></div><div class="table-head"><span>{{ config.entityLabel }}</span><span>类别 / 关联人</span><span>{{ config.dateLabel || '有效期' }}</span><span>{{ config.locationLabel || '存放位置' }}</span><span></span></div><div v-for="item in docs" :key="item.id" class="table-row clickable" role="button" tabindex="0" @click="openDetail(item)" @keydown.enter="openDetail(item)"><div class="item-title"><span class="file-icon"><Archive :size="16" /></span><div><b>{{ item.name }}</b><small>{{ item.number || '未填写编号' }}</small></div></div><span><b>{{ item.category }}</b><small>{{ item.holder }}</small></span><span><b :class="{ dangerText: days(item.expires) < 0, warnText: days(item.expires) >= 0 && days(item.expires) <= 90 }">{{ item.expires }}</b><small>{{ days(item.expires) < 0 ? '已过期' : `${config.remainingPrefix || '剩余'} ${days(item.expires)} 天` }}</small></span><span><b>{{ item.location || '未填写' }}</b><small>{{ config.copyLabel || '备份位置' }}：{{ item.copyLocation || '未填写' }}</small></span><button class="icon-btn" tabindex="-1"><ChevronRight :size="16" /></button></div><div v-if="!docs.length" class="empty">没有匹配的档案。</div></section>
        <section v-else-if="active === 'history'" class="panel"><div class="panel-head"><div><span class="eyebrow">{{ config.historyLabel }}</span><h3>{{ config.historyTitle || `${config.entityLabel}历史` }}</h3></div><span class="count-note">{{ borrowed.length }} 条待处理</span></div><div class="history-grid"><div v-for="entry in state.usages" :key="entry.id" class="history-card"><div class="history-date">{{ entry.date }}<span :class="entry.status === '借用中' ? 'warnText' : 'goodText'">{{ entry.status }}</span></div><b>{{ state.documents.find(x => x.id === entry.documentId)?.name || config.entityLabel }}</b><p>{{ entry.purpose }} · 经办人 {{ entry.borrower }}</p><small>{{ entry.note || '无备注' }}</small></div></div></section>
        <section v-else-if="active === 'insurance'" class="panel"><div class="panel-head"><div><span class="eyebrow">{{ config.secondaryLabel }}</span><h3>{{ config.secondaryTitle || config.secondaryLabel }}</h3></div><button class="text-btn" @click="openCreate">{{ config.secondaryAction || '添加记录' }} <Plus :size="14" /></button></div><div class="policy-grid"><div v-for="item in state.policies" :key="item.id" class="policy-card"><div class="policy-logo"><ShieldCheck :size="20" /></div><div><b>{{ item.type }}</b><small>{{ item.company }} · {{ item.number }}</small></div><span class="policy-date">{{ config.secondaryDatePrefix || '到期' }} {{ item.expires }}</span><strong>{{ item.amount ? `¥${Number(item.amount).toLocaleString()}` : (item.score || '进行中') }} <small>{{ item.payment || item.meta || '' }}</small></strong><p>{{ item.insured || item.description || '已加入家庭计划' }}</p></div></div></section>
        <section v-else class="content-grid"><article class="panel"><div class="panel-head"><div><span class="eyebrow">成长记录</span><h3>成就徽章</h3></div><Award :size="18" class="accent" /></div><div class="badge-grid"><div v-for="(item, index) in config.achievements" :key="item" class="badge" :class="{ unlocked: index < Math.min(5, state.documents.length + 2) }"><span><Award :size="18" /></span><b>{{ item }}</b><small>{{ index < 5 ? '已解锁' : '待解锁' }}</small></div></div></article><article class="panel"><div class="panel-head"><div><span class="eyebrow">家庭榜单</span><h3>档案管理达人</h3></div></div><div class="rank"><b>01</b><span class="rank-avatar">林</span><div><b>林小满</b><small>建档 {{ state.documents.length }} 份 · 处理率 92%</small></div><strong>98</strong></div><div class="rank"><b>02</b><span class="rank-avatar alt">周</span><div><b>周予安</b><small>建档 8 份 · 处理率 88%</small></div><strong>86</strong></div></article></section>
      </div>
    </main>
    <div v-if="showForm" class="modal-backdrop" @click.self="showForm = false"><form class="modal" @submit.prevent="save"><div class="modal-head"><div><span class="eyebrow">建立家庭档案</span><h3>{{ config.form.title }}</h3></div><button type="button" class="icon-btn" @click="showForm = false"><X :size="17" /></button></div><div class="form-grid"><label v-for="[key, label, placeholder, type] in config.form.fields" :key="key" :class="{ wide: key === 'notes' }">{{ label }}<input v-model="draft[key]" :type="type || 'text'" :placeholder="placeholder" /></label><label class="wide upload-field">照片或扫描件（可选）<input type="file" accept="image/*" @change="readFile" /><small>{{ draft.photo ? '已读取图片，可随档案保存' : '支持 JPG、PNG，内容仅保存在本机' }}</small></label></div><div class="modal-actions"><button type="button" class="secondary" @click="showForm = false">取消</button><button class="primary" type="submit"><FilePlus2 :size="16" />保存档案</button></div></form></div>
  </div>
</template>
