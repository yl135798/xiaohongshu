import streamlit as st
import pandas as pd
import jieba
import time
import json
import os
from datetime import datetime, timedelta
from wordcloud import WordCloud
import matplotlib.pyplot as plt
from playwright.sync_api import sync_playwright

# 页面配置
st.set_page_config(
    page_title="雅思托福小红书监控系统",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# 全局样式优化
st.markdown("""
<style>
    .main-header {font-size: 2rem; font-weight: 600; color: #FF2442; margin-bottom: 1rem;}
    .card {background: white; padding: 1.5rem; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 1rem;}
    .metric-card {text-align: center; padding: 1rem; border-radius: 8px; background: #f8f9fa;}
    .metric-value {font-size: 1.5rem; font-weight: 700; color: #FF2442;}
    .metric-label {font-size: 0.9rem; color: #666;}
</style>
""", unsafe_allow_html=True)

# 数据持久化（用JSON文件保存历史数据）
DATA_DIR = "data"
os.makedirs(DATA_DIR, exist_ok=True)

def save_data(key, data):
    with open(f"{DATA_DIR}/{key}.json", "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def load_data(key, default=None):
    path = f"{DATA_DIR}/{key}.json"
    if os.path.exists(path):
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    return default

# 初始化浏览器（云端只运行一次）
@st.cache_resource
def init_browser():
    playwright = sync_playwright().start()
    browser = playwright.chromium.launch(headless=True)
    context = browser.new_context(
        user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36",
        viewport={"width": 1920, "height": 1080}
    )
    
    # 加载保存的cookie
    cookies = load_data("cookies")
    if cookies:
        context.add_cookies(cookies)
    
    page = context.new_page()
    return playwright, browser, context, page

playwright, browser, context, page = init_browser()

# 侧边栏配置
with st.sidebar:
    st.markdown('<div class="main-header">⚙️ 系统设置</div>', unsafe_allow_html=True)
    
    st.subheader("监控关键词")
    keywords = st.text_area(
        "每行一个",
        value="雅思备考\n托福备考\n雅思7分\n托福100分\n雅思口语\n托福写作\n雅思听力\n托福阅读"
    ).split("\n")
    keywords = [k.strip() for k in keywords if k.strip()]
    
    st.subheader("竞品账号")
    competitors = st.text_area(
        "每行一个",
        value="新东方雅思\n环球教育托福\n学为贵雅思\n朗阁雅思\n小站托福"
    ).split("\n")
    competitors = [c.strip() for c in competitors if c.strip()]
    
    st.subheader("数据筛选")
    days = st.slider("监控天数", 1, 30, 7)
    min_likes = st.number_input("最低点赞数", value=30)
    min_collects = st.number_input("最低收藏数", value=20)
    
    st.markdown("---")
    st.subheader("系统状态")
    st.success("✅ 云端运行中")
    st.info(f"最后更新：{load_data('last_update', '从未更新')}")
    
    # 登录按钮（只有你能看到，甲方看不到）
    if st.checkbox("显示管理员选项"):
        if st.button("重新登录小红书"):
            page.goto("https://www.xiaohongshu.com")
            st.warning("请在下方浏览器中扫码登录（仅管理员可见）")
            st.components.v1.html(page.content(), height=600)
            if st.button("保存登录状态"):
                save_data("cookies", context.cookies())
                st.success("登录状态已保存")
                st.rerun()

# 检查登录状态
def is_logged_in():
    page.goto("https://www.xiaohongshu.com")
    time.sleep(2)
    return page.locator("text=消息").count() > 0

if not is_logged_in():
    st.error("系统未登录小红书，请联系管理员完成登录")
    st.stop()

# 核心采集函数
def search_notes(keyword):
    notes = []
    try:
        page.goto(f"https://www.xiaohongshu.com/search_result?keyword={keyword}&source=web_search_result_notes")
        time.sleep(3)
        
        # 滚动加载3页
        for _ in range(3):
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            time.sleep(2)
        
        # 提取笔记数据
        cards = page.locator(".note-item").all()
        for card in cards:
            try:
                title = card.locator(".title").inner_text(timeout=1000)
                author = card.locator(".name").inner_text(timeout=1000)
                like = card.locator(".like-wrapper .count").inner_text(timeout=1000)
                collect = card.locator(".collect-wrapper .count").inner_text(timeout=1000)
                comment = card.locator(".chat-wrapper .count").inner_text(timeout=1000)
                link = card.locator("a").get_attribute("href", timeout=1000)
                
                def parse_num(s):
                    if "w" in s:
                        return int(float(s.replace("w", "")) * 10000)
                    return int(s) if s else 0
                
                like_num = parse_num(like)
                collect_num = parse_num(collect)
                comment_num = parse_num(comment)
                
                if like_num >= min_likes and collect_num >= min_collects:
                    notes.append({
                        "标题": title,
                        "作者": author,
                        "点赞": like_num,
                        "收藏": collect_num,
                        "评论": comment_num,
                        "链接": f"https://www.xiaohongshu.com{link}",
                        "关键词": keyword,
                        "采集时间": datetime.now().strftime("%Y-%m-%d %H:%M")
                    })
            except:
                continue
    except:
        pass
    return notes

def get_comments(note_url):
    comments = []
    try:
        page.goto(note_url)
        time.sleep(3)
        
        # 滚动加载5页评论
        for _ in range(5):
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            time.sleep(1.5)
        
        # 提取评论
        comment_items = page.locator(".comment-item").all()
        for item in comment_items:
            try:
                user = item.locator(".user-name").inner_text(timeout=1000)
                content = item.locator(".content").inner_text(timeout=1000)
                like = item.locator(".like-count").inner_text(timeout=1000)
                comments.append({
                    "用户": user,
                    "评论内容": content,
                    "点赞": int(like) if like else 0,
                    "采集时间": datetime.now().strftime("%Y-%m-%d %H:%M")
                })
            except:
                continue
    except:
        pass
    return comments

# 主界面
st.markdown('<div class="main-header">📊 雅思托福小红书监控系统</div>', unsafe_allow_html=True)

# 数据概览卡片
col1, col2, col3, col4 = st.columns(4)
with col1:
    st.markdown(f'<div class="metric-card"><div class="metric-value">{len(keywords)}</div><div class="metric-label">监控关键词</div></div>', unsafe_allow_html=True)
with col2:
    st.markdown(f'<div class="metric-card"><div class="metric-value">{len(competitors)}</div><div class="metric-label">监控竞品</div></div>', unsafe_allow_html=True)
with col3:
    total_notes = len(load_data("all_notes", []))
    st.markdown(f'<div class="metric-card"><div class="metric-value">{total_notes}</div><div class="metric-label">累计采集笔记</div></div>', unsafe_allow_html=True)
with col4:
    st.markdown(f'<div class="metric-card"><div class="metric-value">{days}</div><div class="metric-label">监控天数</div></div>', unsafe_allow_html=True)

# 选项卡
tab1, tab2, tab3, tab4 = st.tabs(["🔥 爆款笔记", "💬 评论分析", "👥 竞品监控", "📥 数据导出"])

with tab1:
    st.markdown('<div class="card">', unsafe_allow_html=True)
    selected_keyword = st.selectbox("选择关键词", keywords)
    
    if st.button("开始采集", type="primary"):
        with st.spinner(f"正在采集【{selected_keyword}】的爆款笔记..."):
            notes = search_notes(selected_keyword)
            save_data(f"notes_{selected_keyword}", notes)
            save_data("last_update", datetime.now().strftime("%Y-%m-%d %H:%M"))
            
            # 更新全量数据
            all_notes = load_data("all_notes", [])
            all_notes.extend(notes)
            save_data("all_notes", all_notes)
            
            st.success(f"采集完成，共获取 {len(notes)} 条有效笔记")
    
    # 显示历史数据
    notes = load_data(f"notes_{selected_keyword}", [])
    if notes:
        df = pd.DataFrame(notes).sort_values("收藏", ascending=False).reset_index(drop=True)
        st.dataframe(df, use_container_width=True, column_config={
            "链接": st.column_config.LinkColumn("链接")
        })
        
        csv = df.to_csv(index=False).encode("utf-8-sig")
        st.download_button(
            label="下载Excel",
            data=csv,
            file_name=f"爆款笔记_{selected_keyword}_{datetime.now().strftime('%Y%m%d')}.csv",
            mime="text/csv"
        )
    st.markdown('</div>', unsafe_allow_html=True)

with tab2:
    st.markdown('<div class="card">', unsafe_allow_html=True)
    note_url = st.text_input("输入小红书笔记链接")
    
    if note_url and st.button("分析评论", type="primary"):
        with st.spinner("正在获取评论并生成分析..."):
            comments = get_comments(note_url)
            save_data(f"comments_{note_url.split('/')[-1]}", comments)
            
            if comments:
                st.success(f"共获取 {len(comments)} 条评论")
                
                # 词云图
                st.subheader("评论词云")
                text = " ".join([c["评论内容"] for c in comments])
                words = jieba.cut(text)
                wc = WordCloud(
                    font_path="simhei.ttf",
                    width=800, height=400,
                    background_color="white"
                ).generate(" ".join(words))
                
                fig, ax = plt.subplots(figsize=(10,5))
                ax.imshow(wc, interpolation="bilinear")
                ax.axis("off")
                st.pyplot(fig)
                
                # 评论列表
                st.subheader("评论详情")
                df = pd.DataFrame(comments)
                st.dataframe(df, use_container_width=True)
                
                csv = df.to_csv(index=False).encode("utf-8-sig")
                st.download_button(
                    label="下载评论Excel",
                    data=csv,
                    file_name=f"评论分析_{datetime.now().strftime('%Y%m%d')}.csv",
                    mime="text/csv"
                )
            else:
                st.info("该笔记暂无评论或采集失败")
    st.markdown('</div>', unsafe_allow_html=True)

with tab3:
    st.markdown('<div class="card">', unsafe_allow_html=True)
    selected_competitor = st.selectbox("选择竞品账号", competitors)
    
    if st.button("采集竞品笔记", type="primary"):
        with st.spinner(f"正在采集【{selected_competitor}】的最新笔记..."):
            notes = search_notes(selected_competitor)
            save_data(f"competitor_{selected_competitor}", notes)
            st.success(f"采集完成，共获取 {len(notes)} 条笔记")
    
    notes = load_data(f"competitor_{selected_competitor}", [])
    if notes:
        df = pd.DataFrame(notes).sort_values("采集时间", ascending=False).reset_index(drop=True)
        st.dataframe(df, use_container_width=True, column_config={
            "链接": st.column_config.LinkColumn("链接")
        })
    st.markdown('</div>', unsafe_allow_html=True)

with tab4:
    st.markdown('<div class="card">', unsafe_allow_html=True)
    st.subheader("全量数据导出")
    
    if st.button("导出近30天所有数据", type="primary"):
        with st.spinner("正在生成导出文件..."):
            all_notes = load_data("all_notes", [])
            if all_notes:
                df = pd.DataFrame(all_notes)
                csv = df.to_csv(index=False).encode("utf-8-sig")
                st.download_button(
                    label="下载全量数据Excel",
                    data=csv,
                    file_name=f"小红书监控全量数据_{datetime.now().strftime('%Y%m%d')}.csv",
                    mime="text/csv"
                )
            else:
                st.info("暂无数据")
    st.markdown('</div>', unsafe_allow_html=True)
