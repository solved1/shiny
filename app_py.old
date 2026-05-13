from shiny import App, render, ui
import pandas as pd
import plotly.express as px
from datetime import datetime
import subprocess  # Required to execute external scripts

# Define the UI
app_ui = ui.page_navbar(
    # Tab 1: Faculty Availability
    ui.nav_panel("Faculty Availability",
        ui.layout_sidebar(
            ui.sidebar(
                ui.input_file("file1", "Upload RedCap Faculty Schedule Survey CSV file", accept=[".csv"]),
                ui.input_file("file2", "Upload Current Track Affiliation Report CSV file", accept=[".csv"]),
            ),
            ui.h3("Faculty Schedule"),
            ui.output_ui("faculty_schedule_table"),
        ),
    ),
    # Tab 2: Student Availability
    ui.nav_panel("Student Availability",
        ui.layout_sidebar(
            ui.sidebar(
                ui.input_file("student_file", "Upload Student CSV File (Schedule)", accept=[".csv"]),
                ui.input_file("student_details_file", "Upload Student Details CSV File", accept=[".csv"]),
            ),
            ui.h3("Student Availability"),
            ui.output_ui("student_schedule_table"),
        ),
    ),
    # Tab 3: Calculate Matches
    ui.nav_panel("Calculate Matches",
        ui.layout_sidebar(
            ui.sidebar(
                #ui.input_action_button("run_matches", "Run Matching Algorithm"),
                ui.input_file("upload_csv", "Upload Matching Algorithm Results CSV file", accept=[".csv"]),
            ),
            ui.h3("Calculate Matches"),
            ui.output_text("matches_result"),
            ui.output_ui("calendar"),
        ),
    ),
)

# Define the server logic
def server(input, output, session):
    ### Tab 1: Faculty Availability ###
    @output
    @render.ui
    def faculty_schedule_table():
        if not input.file1() and not input.file2():
            return ui.HTML("<p>No files uploaded yet.</p>")
        
        uploaded_files = [input.file1(), input.file2()]
        dfs = []
        for file_info in uploaded_files:
            if file_info:
                file_path = file_info[0]["datapath"]
                # Read CSV files instead of Excel files
                df = pd.read_csv(file_path)
                dfs.append(df)
        
        if not dfs:
            return ui.HTML("<p>No valid files uploaded.</p>")
        combined_df = pd.concat(dfs, ignore_index=True)

        required_columns = ["Last Name", "First Name"]
        missing_columns = [col for col in required_columns if col not in combined_df.columns]
        if missing_columns:
            return ui.HTML(f"<p>Missing columns in Faculty Schedule file: {', '.join(missing_columns)}</p>")

        # Load the second file (Faculty Details)
        if not input.file2():
            return ui.HTML("<p>Faculty details file not uploaded yet.</p>")
        
        file_info_details = input.file2()[0]
        file_path_details = file_info_details["datapath"]
        details_df = pd.read_csv(file_path_details)

        required_details_columns = ["First Name", "Last Name", "Track"]
        missing_details_columns = [col for col in required_details_columns if col not in details_df.columns]
        if missing_details_columns:
            return ui.HTML(f"<p>Missing columns in Faculty Details file: {', '.join(missing_details_columns)}</p>")

        try:
            # Extract columns containing specific dates
            all_date_columns = [col for col in combined_df.columns if "Monday, January 26" in col or "Tuesday, January 27" in col or "Monday, February 2" in col or "Tuesday, February 3" in col]
            unwanted_columns = [
                "Monday, January 26 - BME (choice=Available at all these times)",
                "Monday, January 26 - BME (choice=Not available).1",
                "Tuesday, January 27 - BME (choice=Available at all these times)",
                "Tuesday, January 27 - BME (choice=Not available).1",
                "Monday, February 2 - BME (choice=Available at all these times)",
                "Monday, February 2 - BME (choice=Not available).1",
                "Tuesday, February 3 - BME (choice=Available at all these times)",
                "Tuesday, February 3 - BME (choice=Not available).1",
            ]
            date_columns = [col for col in all_date_columns if col not in unwanted_columns]

            grouped_columns = {}
            for col in date_columns:
                date_str = col.split("-")[0].strip()
                if date_str not in grouped_columns:
                    grouped_columns[date_str] = []
                grouped_columns[date_str].append(col)

            grouped_time_slots = {}
            for date, columns in grouped_columns.items():
                time_slots = []
                for col in columns:
                    if "choice=" in col:
                        time_str = col.split("choice=")[1].strip().lower()
                        time_str = time_str.replace(")", "")
                        time_str = time_str.replace("  ", " ")
                        try:
                            parsed_time = datetime.strptime(time_str, "%I:%M %p").strftime("%I:%M %p")
                            time_slots.append(parsed_time)
                        except ValueError:
                            continue
                grouped_time_slots[date] = time_slots

            faculty_names = combined_df["First Name"].astype(str) + " " + combined_df["Last Name"].astype(str)

            def map_availability(value):
                if value == "Checked":
                    return '<div style="width: 100%; height: 30px; background-color: lightblue; border: 1px solid gray;"></div>'
                elif value == "Unchecked":
                    return '<div style="width: 100%; height: 30px; background-color: whitesmoke; border: 1px solid gray;"></div>'
                else:
                    return '<div style="width: 100%; height: 30px; background-color: white; border: 1px solid gray;"></div>'

            html_table = ""
            for date, time_slots in grouped_time_slots.items():
                html_table += f"<h2 style='text-align: center;'>Schedule for {date}</h2>"
                html_table += "<table style='border-collapse: collapse; width: 100%;'>"
                html_table += "<tr><th style='text-align: center;'>Faculty Name</th>" + "".join([f"<th style='text-align: center;'>{slot}</th>" for slot in time_slots]) + "</tr>"
                for i, name in enumerate(faculty_names):
                    faculty_row = details_df[
                        (details_df["First Name"] == combined_df["First Name"].iloc[i]) &
                        (details_df["Last Name"] == combined_df["Last Name"].iloc[i])
                    ]
                    track = faculty_row["Track"].iloc[0] if not faculty_row.empty else "No Track Found"

                    row = f"<tr><td style='text-align: center;' title='{track}'>{name}</td>"
                    for col in grouped_columns[date]:
                        availability = combined_df[col].iloc[i]
                        row += f"<td style='padding: 0;'>{map_availability(availability)}</td>"
                    row += "</tr>"
                    html_table += row
                html_table += "</table><br>"

            return ui.HTML(html_table)
        except IndexError:
            return ui.HTML("<p>Required columns for the schedule do not exist in the uploaded file.</p>")



    @output
    @render.ui
    def student_schedule_table():
        if not input.student_file():
            return ui.HTML("<p>No student schedule file uploaded yet.</p>")
        if not input.student_details_file():
            return ui.HTML("<p>No student details file uploaded yet.</p>")
        
        student_file_info = input.student_file()
        student_file_path = student_file_info[0]["datapath"]
        student_df = pd.read_csv(student_file_path)

        student_details_file_info = input.student_details_file()
        student_details_file_path = student_details_file_info[0]["datapath"]
        student_details_df = pd.read_csv(student_details_file_path)

        required_schedule_columns = ["last_name_v2", "first_name_v2"]
        missing_schedule_columns = [col for col in required_schedule_columns if col not in student_df.columns]
        if missing_schedule_columns:
            return ui.HTML(f"<p>Missing columns in Student Schedule file: {', '.join(missing_schedule_columns)}</p>")

        required_details_columns = ["First Name", "Last Name", "Track"]
        missing_details_columns = [col for col in required_details_columns if col not in student_details_df.columns]
        if missing_details_columns:
            return ui.HTML(f"<p>Missing columns in Student Details file: {', '.join(missing_details_columns)}</p>")

        try:
            # Extract columns containing specific dates
            all_date_columns = [col for col in student_df.columns if "jan26___" in col or "jan27___" in col or "feb02___" in col or "feb03___" in col]

            grouped_columns = {}
            for col in all_date_columns:
                date_str = col.split("___")[0].strip()
                if date_str not in grouped_columns:
                    grouped_columns[date_str] = []
                grouped_columns[date_str].append(col)

            grouped_start_times = {}
            for date, columns in grouped_columns.items():
                start_times = []
                for col in columns:
                    time_str = col.split("___")[1].strip().lower()
                    try:
                        start_time = time_str.split("_")[0]
                        if "p" in time_str:
                            if len(start_time) == 3 or len(start_time) == 4:
                                start_time = datetime.strptime(start_time, "%I%M").strftime("%I:%M %p")
                            elif len(start_time) == 1 or len(start_time) == 2:
                                start_time = datetime.strptime(start_time, "%I").strftime("%I:%M %p")
                            start_time = start_time.replace("AM", "PM")
                        elif "a" in time_str:
                            if len(start_time) == 3 or len(start_time) == 4:
                                start_time = datetime.strptime(start_time, "%I%M").strftime("%I:%M %p")
                            elif len(start_time) == 1 or len(start_time) == 2:
                                start_time = datetime.strptime(start_time, "%I").strftime("%I:%M %p")
                        else:
                            raise ValueError("Invalid start time format")
                        start_times.append(start_time)
                    except ValueError:
                        continue
                grouped_start_times[date] = start_times

            names = student_df["first_name_v2"].astype(str) + " " + student_df["last_name_v2"].astype(str)

            def map_availability(value):
                if value == 0:
                    return '<div style="width: 100%; height: 30px; background-color: lightblue; border: 1px solid gray;"></div>'
                elif value == 1:
                    return '<div style="width: 100%; height: 30px; background-color: whitesmoke; border: 1px solid gray;"></div>'
                else:
                    return '<div style="width: 100%; height: 30px; background-color: white; border: 1px solid gray;"></div>'

            html_table = ""
            for date, start_times in grouped_start_times.items():
                html_table += f"<h2 style='text-align: center;'>Schedule for {date}</h2>"
                html_table += "<table style='border-collapse: collapse; width: 100%;'>"
                html_table += "<tr><th style='text-align: center;'>Name</th>" + "".join([f"<th style='text-align: center;'>{start_time}</th>" for start_time in start_times]) + "</tr>"
                for i, name in enumerate(names):
                    student_row = student_details_df[
                        (student_details_df["First Name"] == student_df["first_name_v2"].iloc[i]) &
                        (student_details_df["Last Name"] == student_df["last_name_v2"].iloc[i])
                    ]
                    track = student_row["Track"].iloc[0] if not student_row.empty else "No Track Found"

                    row = f"<tr><td style='text-align: center;' title='{track}'>{name}</td>"
                    for col in grouped_columns[date]:
                        availability = student_df[col].iloc[i]
                        row += f"<td style='padding: 0;'>{map_availability(availability)}</td>"
                    row += "</tr>"
                    html_table += row
                html_table += "</table><br>"

            return ui.HTML(html_table)
        except IndexError:
            return ui.HTML("<p>Required columns for the schedule do not exist in the uploaded file.</p>")
    @output
    @render.text
    def matches_result():
        if input.run_matches():
            try:
                print("DEBUG: Attempting to run schedule_matches_3.py...")
                result = subprocess.run(["python", "../schedule_matches_3.py"], capture_output=True, text=True)
                # Remove <p> and <pre> tags by returning plain text
                return f"Matches calculation completed: {result.stdout.strip()}"
            except Exception as e:
                print(f"DEBUG: Error occurred while running the script: {str(e)}")
                return f"Error running the script: {str(e)}"
    @output
    @render.ui
    def calendar():
        if not input.upload_csv():
            return ui.HTML("<p>No CSV file uploaded yet.</p>")
        
        file_info = input.upload_csv()
        file_path = file_info[0]["datapath"]
        
        try:
            df = pd.read_csv(file_path)
            required_columns = [
                "student_first_name", "student_last_name", "student_track",
                "faculty_first_name", "faculty_last_name", "faculty_appointment",
                "faculty_track", "matching_availability_slot", "is_required_handler", "preference_rank"
            ]
            missing_columns = [col for col in required_columns if col not in df.columns]
            if missing_columns:
                return ui.HTML(f"<p>Missing columns in uploaded file: {', '.join(missing_columns)}</p>")

            # Parse the availability slot into start and end times
            def parse_time_range(time_range):
                try:
                    start, end = time_range.split(" to ")
                    start_time = pd.to_datetime(start, format="%b %d %I:%M %p", errors="coerce")
                    end_time = pd.to_datetime(end, format="%I:%M %p", errors="coerce")
                    return start_time, end_time
                except Exception as e:
                    print(f"DEBUG: Failed to parse time range '{time_range}': {e}")
                    return None, None

            df['start_time'], df['end_time'] = zip(*df['matching_availability_slot'].apply(parse_time_range))

            # Handle invalid rows with missing or incorrect time ranges
            invalid_rows = df[df['start_time'].isnull() | df['end_time'].isnull()]
            if not invalid_rows.empty:
                return ui.HTML(
                    f"<p>Error: Some rows have invalid time ranges. Please check the following rows:</p>"
                    f"<pre>{invalid_rows[['matching_availability_slot']].to_string(index=False)}</pre>"
                )

            # Sort the time slots by start time
            df = df.sort_values(by='start_time')
            unique_slots = df['matching_availability_slot'].drop_duplicates().tolist()
            unique_students = df[['student_last_name', 'student_first_name', 'student_track']].drop_duplicates()

            # Build the HTML table
            html_table = "<table style='border-collapse: collapse; width: 100%; min-width: 250px; border: 1px solid black;'>"
            #html_table += f"<tr><td style='border: 1px solid black; min-width: 250px;' title='{student_track}'>{student_name}</td>"
            html_table += "<tr><th style='border: 1px solid black; min-width: 250px;'>Student Name</th>"

            # Add time slots as column headers
            for slot in unique_slots:
                html_table += f"<th style='border: 1px solid black;'>{slot}</th>"
            html_table += "</tr>"

            # Add rows for each student
            for _, student_row in unique_students.iterrows():
                student_name = f"{student_row['student_last_name']}, {student_row['student_first_name']}"
                student_track = student_row['student_track']

                html_table += f"<tr><td style='border: 1px solid black;' title='{student_track}'>{student_name}</td>"

                for slot in unique_slots:
                    # Filter the dataframe for the current student and slot
                    matching_row = df[
                        (df['student_last_name'] == student_row['student_last_name']) &
                        (df['student_first_name'] == student_row['student_first_name']) &
                        (df['matching_availability_slot'] == slot)
                    ]

                    if not matching_row.empty:
                        # Extract faculty name and tooltip text (appointment and track)
                        faculty_last_name = matching_row.iloc[0]['faculty_last_name'].title()
                        faculty_first_name = matching_row.iloc[0]['faculty_first_name'].title()
                        faculty_name = f"<b>{faculty_last_name}, {faculty_first_name}</b>"  # Add bold and capitalize
                        #faculty_name = f"{matching_row.iloc[0]['faculty_last_name']}, {matching_row.iloc[0]['faculty_first_name']}"
                        faculty_appointment = matching_row.iloc[0]['faculty_appointment']
                        faculty_track = matching_row.iloc[0]['faculty_track']
                        tooltip_text = f"{faculty_appointment}, {faculty_track}"

                        # Handle empty values in 'is_required_handler'
                        is_required_handler = matching_row.iloc[0]['is_required_handler']
                        handler_status = is_required_handler if pd.notna(is_required_handler) else ""

                        # Determine preference text based on preference_rank
                        preference_rank = matching_row.iloc[0]['preference_rank']
                        if 1 <= preference_rank <= 5:
                            preference_text = "Student Preference"
                        elif 101 <= preference_rank <= 110:
                            preference_text = "Handler Preference"
                        elif preference_rank > 110:
                            preference_text = "Program Preference"
                        else:
                            preference_text = ""

                        # Add tooltip and format the content
                        cell_content = (
                            f"<div style='background-color: lightblue; padding: 5px; border: 1px solid black; min-width: 250px; text-align: center;' title='{tooltip_text}'>"
                            f"{faculty_name}<br>{handler_status}<br>{preference_text}</div>"
                        )
                    else:
                        # Empty cell with three lines (second and third lines are blank)
                        cell_content = "<div style='background-color: white; padding: 5px; border: 1px solid black; min-width: 250px; text-align: center;'><br><br><br></div>"

                    html_table += f"<td style='border: 1px solid black;'>{cell_content}</td>"
                html_table += "</tr>"

            html_table += "</table>"

            return ui.HTML(html_table)

        except Exception as e:
            return ui.HTML(f"<p>Error processing file: {e}</p>")

# Create the app
app = App(app_ui, server)

if __name__ == "__main__":
    app.run()