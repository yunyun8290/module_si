[module_tool.py](https://github.com/user-attachments/files/25414830/module_tool.py)
import streamlit as st
import itertools
import json

# ------------------------
# ページタイトル
# ------------------------
st.title("🛠 モジュールシミュレーター")

# ------------------------
# 初期化
# ------------------------
if "modules" not in st.session_state:
    st.session_state.modules = []

# ------------------------
# モジュール選択肢
# ------------------------
module_types = ["攻撃型", "防御型", "支援型"]
rarities = ["紫", "金"]
status_options = {
    "赤": ["筋力強化","敏捷強化","知力強化","精鋭打撃","特攻ダメージ強化","極ダメージ増強","極適応力"],
    "黄": ["攻撃速度","幸運","詠唱","会心","極幸運会心","極HP変動","極HP吸収"],
    "青": ["物理耐性","魔法耐性","極絶境守護"],
    "緑": ["特攻回復強化","マスタリー回復強化","極HP凝縮","極応急処置"]
}

# ------------------------
# ステータスレベル計算関数
# ------------------------
def calc_level(val):
    if val >= 20:
        return 6
    elif 16 <= val <= 19:
        return 5
    elif 12 <= val <= 15:
        return 4
    elif 8 <= val <= 11:
        return 3
    elif 4 <= val <= 7:
        return 2
    elif 1 <= val <= 3:
        return 1
    else:
        return 0

# ------------------------
# モジュール追加フォーム
# ------------------------
with st.form("module_form"):
    col1, col2 = st.columns(2)
    with col1:
        input_name = st.selectbox("タイプ", module_types)
        input_rarity = st.selectbox("レアリティ", rarities)
    with col2:
        st.markdown("**ステータスと値**")
        status_selections = []
        value_selections = []
        for i in range(3):
            status = st.selectbox(f"ステータス{i+1}", [""] + sum(status_options.values(), []), key=f"status_{i}")
            value = st.selectbox(f"数値{i+1}", list(range(11)), key=f"value_{i}")
            status_selections.append(status)
            value_selections.append(value)

    submitted = st.form_submit_button("モジュール追加")

if submitted:
    mod = {
        "type": input_name,
        "rarity": input_rarity,
        "status": {s:v for s,v in zip(status_selections,value_selections) if s}
    }
    st.session_state.modules.append(mod)
    st.success("モジュール追加完了！")

# ------------------------
# JSON保存・読み込み
# ------------------------
st.write("### モジュール保存/読み込み")
col_save, col_load = st.columns(2)
with col_save:
    if st.button("保存用JSONを作成"):
        json_data = json.dumps(st.session_state.modules, ensure_ascii=False, indent=2)
        st.download_button("ダウンロード", data=json_data, file_name="modules.json", mime="application/json")

with col_load:
    uploaded_file = st.file_uploader("JSONファイルから読み込む", type="json")
    if uploaded_file:
        try:
            loaded_modules = json.load(uploaded_file)
            st.session_state.modules = loaded_modules
            st.success("JSONからモジュールを復元しました")
        except Exception as e:
            st.error(f"読み込み失敗: {e}")

# ------------------------
# モジュール一覧表示・削除
# ------------------------
st.write("### 登録モジュール")
to_delete_idx = None
for idx, mod in enumerate(st.session_state.modules):
    mod_status = [f"{k}: {v}" for k,v in mod['status'].items()]
    col1, col2 = st.columns([4,1])
    with col1:
        st.write(f"{idx+1}. {mod['type']} ({mod['rarity']}) - " + " | ".join(mod_status))
    with col2:
        if st.button("削除", key=f"del_{idx}"):
            to_delete_idx = idx

if to_delete_idx is not None:
    st.session_state.modules.pop(to_delete_idx)
    st.info("削除しますか？")

# ------------------------
# 最適化
# ------------------------
st.write("### 最適化候補")
selected_opt_status = st.multiselect("優先ステータス（順序をつける場合は左から優先）", sum(status_options.values(), []))

def generate_combinations(modules, n=4):
    n = min(n, len(modules))
    return list(itertools.combinations(modules, n))

all_combos = generate_combinations(st.session_state.modules, n=4)
combo_list = []
for combo in all_combos:
    total_score_dict = {}
    for mod in combo:
        for s,v in mod['status'].items():
            total_score_dict[s] = total_score_dict.get(s,0) + v

    # 優先ステータス順にレベルタプル作成（順序優先）
    if selected_opt_status:
        priority_levels = tuple(calc_level(total_score_dict.get(s,0)) for s in selected_opt_status)
    else:
        priority_levels = (0,)

    combo_list.append({
        "combo": combo,
        "priority_levels": priority_levels,
        "total_score_dict": total_score_dict
    })

# 優先ステータス順でソートして上位3
combo_list = sorted(combo_list, key=lambda x: x["priority_levels"], reverse=True)[:3]

# ------------------------
# CSS枠付き候補表示（背景透明）
# ------------------------
for idx, item in enumerate(combo_list, 1):
    combo = item["combo"]
    total_score_dict = item["total_score_dict"]

    if idx == 1:
        border_color = "#DAA520"  # 金
    elif idx == 2:
        border_color = "#C0C0C0"  # 銀
    else:
        border_color = "#CD7F32"  # 銅

    st.markdown(f"""
    <div style="border:3px solid {border_color}; padding:10px; margin-bottom:10px; border-radius:10px; background-color:transparent;">
        <b>候補 {idx}</b>
        <ul>
    """, unsafe_allow_html=True)

    for i, mod in enumerate(combo):
        mod_status = [f"{s}: {v}" for s,v in mod['status'].items()]
        st.markdown(f"<li>{i+1}. {mod['type']} ({mod['rarity']}) - " + " | ".join(mod_status) + "</li>", unsafe_allow_html=True)

    st.markdown("<p>ステータス合計値（レベル）:</p>", unsafe_allow_html=True)
    for s,v in sorted(total_score_dict.items(), key=lambda x:x[1], reverse=True):
        level = calc_level(v)
        color = ""
        if s in selected_opt_status:
            if level == 6:
                color = "red"
            elif level == 5:
                color = "blue"
        st.markdown(f"<span style='color:{color}'>{s}: {v} ({level})</span><br>", unsafe_allow_html=True)

    st.markdown("</ul></div>", unsafe_allow_html=True)
