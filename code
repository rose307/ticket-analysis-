import streamlit as st
import pandas as pd
import numpy as np
import io
import os

START_YEAR = 2023
END_YEAR = 2036
MONTHS = ["APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC", "JAN", "FEB", "MAR"]
CATEGORIES = [
    "SUBURBAN SEASON TICKET",
    "SUBURBAN JOURNEY TICKET",
    "NON SUBURBAN SEASON TICKET",
    "NON SUBURBAN JOURNEY TICKET",
    "ATVM",
    "UTS",
    "MOBILE"
]

SAVE_FOLDER = "saved_data"

def save_input_data(category, year_label, df):
    if not os.path.exists(SAVE_FOLDER):
        os.makedirs(SAVE_FOLDER)
    file_path = os.path.join(SAVE_FOLDER, f"{category}_{year_label}.csv")
    df.to_csv(file_path, index=False)

def load_saved_data(category, year_label):
    file_path = os.path.join(SAVE_FOLDER, f"{category}_{year_label}.csv")
    if os.path.exists(file_path):
        return pd.read_csv(file_path)
    else:
        return None

def load_initial_data():
    raw_df = pd.read_csv("data.csv", header=None)
    initial_data = {}
    for category in CATEGORIES:
        cat_idx = raw_df[0].str.contains(category, na=False).idxmax()
        df_part = raw_df.iloc[cat_idx+3:cat_idx+15, 0:4]
        df_part.columns = ["Month", "TKT", "PSG", "AMT"]
        df_part["Month"] = df_part["Month"].str.strip()
        for col in ["TKT", "PSG", "AMT"]:
            df_part[col] = pd.to_numeric(df_part[col], errors='coerce').fillna(0).astype(int)
        df_part = df_part.reset_index(drop=True)
        initial_data[category] = df_part
    return initial_data

def get_saved_or_default_df(category, year_label):
    loaded_df = load_saved_data(category, year_label)
    if loaded_df is not None:
        return loaded_df
    else:
        return pd.DataFrame({
            "Month": MONTHS,
            "TKT": [0]*len(MONTHS),
            "PSG": [0]*len(MONTHS),
            "AMT": [0]*len(MONTHS)
        })

def calc_variation(current, previous):
    with np.errstate(divide='ignore', invalid='ignore'):
        var = (current - previous) / previous * 100
        var = var.replace([np.inf, -np.inf], 0)
        var = var.fillna(0)
    return var.round(2)

def create_comparative_table(prev_df, curr_df, prev_year_label, curr_year_label):
    months = curr_df["Month"]
    comp_data = {"Month": months}
    for col in ["TKT", "PSG", "AMT"]:
        prev_vals = prev_df[col]
        curr_vals = curr_df[col]
        var_vals = calc_variation(curr_vals, prev_vals)
        comp_data[f"{col} Previous Year ({prev_year_label})"] = prev_vals
        comp_data[f"{col} Current Year ({curr_year_label})"] = curr_vals
        comp_data[f"{col} VAR%"] = var_vals
    return pd.DataFrame(comp_data)

def create_cumulative_table(category, comp_df, prev_year_label, curr_year_label):
    months = comp_df["Month"]
    cum_data = {"Month": months}
    for col_prefix in ["TKT", "PSG", "AMT"]:
        prev_year_vals = comp_df[f"{col_prefix} Previous Year ({prev_year_label})"].to_list()
        curr_year_vals = comp_df[f"{col_prefix} Current Year ({curr_year_label})"].to_list()
        prev_cum = []
        curr_cum = []
        for i in range(len(months)):
            if i == 0:
                prev_cum.append(prev_year_vals[i])
                curr_cum.append(curr_year_vals[i])
            else:
                prev_cum.append(prev_cum[-1] + prev_year_vals[i])
                curr_cum.append(curr_cum[-1] + curr_year_vals[i])
        var_cum = calc_variation(pd.Series(curr_cum), pd.Series(prev_cum))
        cum_data[f"{col_prefix} Previous Year ({prev_year_label})"] = prev_cum
        cum_data[f"{col_prefix} Current Year ({curr_year_label})"] = curr_cum
        cum_data[f"{col_prefix} VAR%"] = var_cum
    return pd.DataFrame(cum_data)

def create_excel_download_one_sheet(tables_dict):
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
        row_pos = 0
        workbook = writer.book
        worksheet = workbook.add_worksheet("Summary")
        writer.sheets["Summary"] = worksheet
        for key in tables_dict:
            df = tables_dict[key]
            # Write header
            worksheet.write(row_pos, 0, key.replace("-", " ").title())
            row_pos += 1
            df.to_excel(writer, sheet_name="Summary", startrow=row_pos, startcol=0, index=False)
            row_pos += len(df) + 3
    return output.getvalue()

def main():
    st.title("Yearly Ticket Data Input and Comparison")
    # Load initial data CSV for 2022-23
    if "initial_data_loaded" not in st.session_state:
        st.session_state.initial_data = load_initial_data()
        st.session_state.initial_data_loaded = True

    # Year selection
    year_options = [f"{y}-{str(y+1)[-2:]}" for y in range(START_YEAR, END_YEAR+1)]
    selected_year = st.selectbox("Select Year", year_options)
    selected_year_index = year_options.index(selected_year)

    # Determine previous year label and data
    if selected_year_index == 0:
        prev_year_label = "2022-23"
        prev_year_data = st.session_state.initial_data
    else:
        prev_year_label = year_options[selected_year_index - 1]
        prev_year_data = {}
        for cat in CATEGORIES:
            prev_year_data[cat] = get_saved_or_default_df(cat, prev_year_label)
    
    curr_year_label = selected_year

    comparative_tables = {}
    cumulative_tables = {}

    # For each category, create input table + save button + comparative and cumulative tables
    for category in CATEGORIES:
        st.write(f"## {category}")
        df_key = f"{category}_{curr_year_label}_df"
        # Get current data dataframe, load saved or default
        curr_df = get_saved_or_default_df(category, curr_year_label)

        # Editable table with unique key (not saved in session state because we reload saved on each run)
        edited_df = st.data_editor(curr_df, key=f"{category}_{curr_year_label}_editor", num_rows="dynamic")

        # Save button
        if st.button(f"Save Entry for {category} ({curr_year_label})"):
            save_input_data(category, curr_year_label, edited_df)
            st.success(f"Saved {category} data for {curr_year_label}")
            curr_df = edited_df  # Update current df to saved one

        # Load previous year data -> fallback to zeros if not found (handled in get_saved_or_default_df)
        prev_df = prev_year_data.get(category, get_saved_or_default_df(category, prev_year_label))

        # Ensure month order
        prev_df = prev_df.set_index("Month").reindex(MONTHS).reset_index()
        curr_df = curr_df.set_index("Month").reindex(MONTHS).reset_index()

        # Create and show comparative and cumulative tables
        comp_df = create_comparative_table(prev_df, curr_df, prev_year_label, curr_year_label)
        cum_df = create_cumulative_table(category, comp_df, prev_year_label, curr_year_label)
        
        st.write(f"### Comparative Table for {category}")
        st.dataframe(comp_df, use_container_width=True)

        st.write(f"### Cumulative Table for {category}")
        st.dataframe(cum_df, use_container_width=True)

        # Store tables for download
        comparative_tables[f"{category}-comp"] = comp_df
        cumulative_tables[f"{category}-cum"] = cum_df

    # Combined Excel download for all tables
    st.write("---")
    excel_data = create_excel_download_one_sheet({**comparative_tables, **cumulative_tables})
    st.download_button(
        label="Download Combined Comparative and Cumulative Excel",
        data=excel_data,
        file_name=f"ticket_data_{curr_year_label}.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    )


if __name__ == "__main__":
    main()
