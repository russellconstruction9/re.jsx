from flask import Flask, render_template, request, send_file
import pandas as pd
from fpdf import FPDF

app = Flask(__name__)

# Constants
COST_PER_SET = 2200  # Cost per set of spray foam
BOARD_FEET_PER_SET = {"open-cell": 18000, "closed-cell": 4000}
LABOR_COST_PER_SQFT = {"open-cell": 0.50, "closed-cell": 1.25}

# Function to calculate spray foam estimate
def calculate_spray_foam_estimate(length, width, height, insulation_type, location_factor=1.0):
    if insulation_type not in BOARD_FEET_PER_SET:
        raise ValueError("Invalid insulation type. Choose 'open-cell' or 'closed-cell'.")

    square_footage = length * width
    board_feet_needed = square_footage * height
    sets_needed = board_feet_needed / BOARD_FEET_PER_SET[insulation_type]
    material_cost = sets_needed * COST_PER_SET
    labor_cost = (square_footage * LABOR_COST_PER_SQFT[insulation_type]) * location_factor
    total_cost = material_cost + labor_cost

    return {
        "Length (ft)": length,
        "Width (ft)": width,
        "Height (ft)": height,
        "Square Footage": square_footage,
        "Board Feet Needed": board_feet_needed,
        "Sets Needed": round(sets_needed, 2),
        "Material Cost ($)": f"${round(material_cost, 2):,.2f}",
        "Labor Cost ($)": f"${round(labor_cost, 2):,.2f}",
        "Total Cost ($)": f"${round(total_cost, 2):,.2f}",
    }

# Function to generate a PDF quote
def generate_quote_pdf(estimate_data):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", "B", 16)
    pdf.cell(200, 10, "Spray Foam Insulation Estimate", ln=True, align="C")
    pdf.ln(10)
    pdf.set_font("Arial", "", 12)

    for key, value in estimate_data.items():
        pdf.cell(80, 10, f"{key}:", border=1)
        pdf.cell(100, 10, f"{value}", border=1, ln=True)

    pdf.ln(10)
    pdf.cell(200, 10, "Thank you for using our spray foam estimator!", ln=True, align="L")

    pdf_filename = "Spray_Foam_Estimate.pdf"
    pdf.output(pdf_filename)
    return pdf_filename

# Flask Routes
@app.route("/", methods=["GET", "POST"])
def home():
    estimate_result = None
    error_message = None

    if request.method == "POST":
        try:
            length = float(request.form["length"])
            width = float(request.form["width"])
            height = float(request.form["height"])
            insulation_type = request.form["insulation_type"]
            location_factor = float(request.form["location_factor"])

            estimate_result = calculate_spray_foam_estimate(length, width, height, insulation_type, location_factor)

        except ValueError as e:
            error_message = str(e)

    return render_template("index.html", estimate=estimate_result, error=error_message)

@app.route("/download_pdf")
def download_pdf():
    pdf_filename = generate_quote_pdf(request.args)
    return send_file(pdf_filename, as_attachment=True)

if __name__ == "__main__":
    app.run(debug=True)
