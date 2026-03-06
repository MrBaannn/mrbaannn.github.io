<template>
  <div class="home-page">
    <!-- Header Section -->
    <header class="hero-section">
      <h1 class="hero-title">欢迎来到般般的博客</h1>
      <p class="hero-subtitle">
        投资有风险，理财需谨慎<br/>
        本页面所展示的持仓信息仅为个人记录，不构成任何投资建议，仅供参考
      </p>
    </header>

    <!-- Profile + NAV Chart Section -->
    <section class="profile-chart-section">
      <div class="profile-card">
        <div class="avatar-wrapper">
          <img src="/images/tang.jpg" alt="般般的头像" class="avatar" />
        </div>
        <div class="profile-info">
          <h2 class="profile-name">般般</h2>
          <p class="profile-desc">你好，我是般般 :-)</p>
        </div>
      </div>
      <div class="chart-card">
        <h3 class="section-title">📈 资产走势</h3>
        <div class="chart-container">
          <NavChart :data="navData" />
        </div>
      </div>
    </section>

    <!-- Holdings Section -->
    <section class="holdings-section">
      <h3 class="section-title" style="text-align: center;">📊 持仓分布</h3>
      <div class="pie-container">
        <HoldingsPie :allData="allHoldingsData" />
      </div>
    </section>

    <!-- Footer -->
    <footer class="page-footer">
      <p>© {{ new Date().getFullYear() }} 般般的博客 · Powered by VitePress</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import NavChart from './NavChart.vue'
import HoldingsPie from './HoldingsPie.vue'

const navData = ref([])
const allHoldingsData = ref([])

// Parse YYMMDD filename to readable date
function parseDate(filename) {
  const name = filename.replace('.json', '')
  const yy = name.substring(0, 2)
  const mm = name.substring(2, 4)
  const dd = name.substring(4, 6)
  return `20${yy}-${mm}-${dd}`
}

// Sort key for filenames
function dateSortKey(filename) {
  const name = filename.replace('.json', '')
  return parseInt(name)
}

onMounted(async () => {
  try {
    // Fetch the data index file that lists all available data files
    const indexResp = await fetch('/data/index.json')
    const fileList = await indexResp.json()

    // Sort files chronologically
    const sorted = fileList.sort((a, b) => dateSortKey(a) - dateSortKey(b))

    // Load all data files
    const allData = []
    for (const file of sorted) {
      try {
        const resp = await fetch(`/data/${file}`)
        const data = await resp.json()
        allData.push({
          file,
          date: parseDate(file),
          nav: parseFloat(data.nav),
          categories: data.categories || []
        })
      } catch (e) {
        console.warn(`Failed to load ${file}:`, e)
      }
    }

    // Build nav chart data
    navData.value = allData.map(d => ({
      date: d.date,
      nav: d.nav
    }))

    // Pass all holdings data (newest first for date selector)
    allHoldingsData.value = allData.slice().reverse()
  } catch (e) {
    console.error('Failed to load portfolio data:', e)
  }
})
</script>

<style scoped>
.home-page {
  max-width: none;
  margin: 0 auto;
  padding: 40px 5vw 60px;
  font-family: 'Noto Sans SC', sans-serif;
}

/* Hero */
.hero-section {
  text-align: center;
  margin-bottom: 48px;
  padding: 48px 20px 36px;
}

.hero-title {
  font-size: 2rem;
  font-weight: 700;
  color: var(--color-text-primary);
  margin: 0 0 12px;
  letter-spacing: -0.02em;
}

.hero-subtitle {
  font-size: 0.85rem;
  color: var(--color-text-muted);
  max-width: 520px;
  margin: 0 auto;
  line-height: 1.7;
}

/* Profile + Chart */
.profile-chart-section {
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: 24px;
  margin-bottom: 48px;
  align-items: stretch;
}

.profile-card {
  background: var(--color-card);
  border: 1px solid var(--color-border);
  border-radius: var(--radius);
  padding: 32px 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.3s ease;
}

.profile-card:hover {
  box-shadow: var(--shadow-md);
}

.avatar-wrapper {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  overflow: hidden;
  margin-bottom: 20px;
  border: 3px solid var(--color-border);
  box-shadow: var(--shadow-sm);
}

.avatar {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.profile-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.profile-name {
  font-size: 1.3rem;
  font-weight: 600;
  color: var(--color-text-primary);
  margin: 0 0 6px;
}

.profile-desc {
  font-size: 0.85rem;
  color: var(--color-text-secondary);
  margin: 0;
}

.chart-card {
  background: var(--color-card);
  border: 1px solid var(--color-border);
  border-radius: var(--radius);
  padding: 28px;
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.3s ease;
  display: flex;
  flex-direction: column;
}

.chart-card:hover {
  box-shadow: var(--shadow-md);
}

.section-title {
  font-size: 1.05rem;
  font-weight: 600;
  color: var(--color-text-primary);
  margin: 0 0 20px;
}

.chart-container {
  flex: 1;
  min-height: 280px;
}

/* Holdings */
.holdings-section {
  background: var(--color-card);
  border: 1px solid var(--color-border);
  border-radius: var(--radius);
  padding: 32px;
  box-shadow: var(--shadow-sm);
  margin-bottom: 48px;
}

.pie-container {
  max-width: 100%;
  margin: 0 auto;
}

/* Footer */
.page-footer {
  text-align: center;
  padding: 24px 0;
  border-top: 1px solid var(--color-border);
}

.page-footer p {
  font-size: 0.8rem;
  color: var(--color-text-muted);
  margin: 0;
}

/* Responsive */
@media (max-width: 768px) {
  .home-page {
    padding: 24px 16px 40px;
  }

  .hero-title {
    font-size: 1.5rem;
  }

  .profile-chart-section {
    grid-template-columns: 1fr;
  }

  .avatar-wrapper {
    width: 90px;
    height: 90px;
  }

  .chart-container {
    min-height: 220px;
  }
}
</style>
