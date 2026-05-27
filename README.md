# @title 📊 Genomic Risk & Clinical Analysis Dashboard {display-mode: "form"}
import pandas as pd
import numpy as np
import json
from IPython.display import HTML
from google.colab import output

# --- 1. Data Preparation ---
# We rely on variables already in memory: df_metrics, df_dist_merged, df_genotypes_large, prs_corr

def get_dashboard_data():
    # Model Performance
    perf_data = df_metrics.to_dict(orient='records')
    
    # PRS Distribution
    prs_values = df_genotypes_large['PRS_Z'].tolist()
    
    # Risk Tiers
    risk_counts = df_genotypes_large['Risk_Tier'].value_counts().to_dict()
    
    # Clinical Correlation (Top 5)
    corr_data = prs_corr.reset_index().rename(columns={'index': 'Metric', 'PRS_Z': 'Correlation'}).to_dict(orient='records')

    return {
        "performance": perf_data,
        "prs_dist": prs_values,
        "risk_tiers": risk_counts,
        "correlations": corr_data
    }

dashboard_json = json.dumps(get_dashboard_data())

def _report_js_error(message):
    print(f"JavaScript Error: {message}")

output.register_callback('report_js_error', _report_js_error)

# --- 2. HTML/JS/CSS Dashboard ---
html_code = """
<!DOCTYPE html>
<html>
<head>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f6f8; margin: 0; padding: 20px; }
        .dashboard-container { display: flex; flex-direction: column; gap: 20px; }
        .kpi-row { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; }
        .card { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }
        .kpi-card { text-align: center; }
        .kpi-value { font-size: 24px; font-weight: bold; color: #2c3e50; margin-top: 5px; }
        .kpi-label { font-size: 14px; color: #7f8c8d; text-transform: uppercase; }
        .main-row { display: grid; grid-template-columns: 2fr 1fr; gap: 20px; }
        .bottom-row { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
        .canvas-wrapper { position: relative; flex-grow: 1; min-height: 300px; }
        h3 { margin-top: 0; color: #34495e; font-size: 16px; border-bottom: 1px solid #eee; padding-bottom: 10px; }
    </style>
</head>
<body>
    <div class="dashboard-container">
        <div class="kpi-row">
            <div class="card kpi-card">
                <div class="kpi-label">Total Samples</div>
                <div class="kpi-value" id="stat-n">0</div>
            </div>
            <div class="card kpi-card">
                <div class="kpi-label">Max AUC</div>
                <div class="kpi-value" id="stat-auc">0.00</div>
            </div>
            <div class="card kpi-card">
                <div class="kpi-label">High Risk Group</div>
                <div class="kpi-value" id="stat-high-risk">0%</div>
            </div>
            <div class="card kpi-card">
                <div class="kpi-label">Avg Genotype (SNP_44)</div>
                <div class="kpi-value">0.18</div>
            </div>
        </div>

        <div class="main-row">
            <div class="card">
                <h3>PRS Score Distribution</h3>
                <div class="canvas-wrapper"><canvas id="prsChart"></canvas></div>
            </div>
            <div class="card">
                <h3>Risk Stratification</h3>
                <div class="canvas-wrapper"><canvas id="tierChart"></canvas></div>
            </div>
        </div>

        <div class="bottom-row">
            <div class="card">
                <h3>Model Comparison (AUC)</h3>
                <div class="canvas-wrapper"><canvas id="modelChart"></canvas></div>
            </div>
            <div class="card">
                <h3>PRS-Clinical Correlation</h3>
                <div class="canvas-wrapper"><canvas id="corrChart"></canvas></div>
            </div>
        </div>
    </div>

    <script>
        window.onerror = function(msg) { google.colab.kernel.invokeFunction('report_js_error', [msg], {}); };

        const data = DATA_PLACEHOLDER;

        // Update KPIs
        document.getElementById('stat-n').innerText = data.prs_dist.length;
        const aucs = data.performance.filter(d => d.Metric === 'AUC').map(d => d.Value);
        document.getElementById('stat-auc').innerText = Math.max(...aucs).toFixed(3);
        const highRiskCount = data.risk_tiers['High Risk (> 1 SD)'] || 0;
        document.getElementById('stat-high-risk').innerText = ((highRiskCount / data.prs_dist.length) * 100).toFixed(1) + '%';

        // PRS Distribution Histogram
        new Chart(document.getElementById('prsChart'), {
            type: 'bar',
            data: {
                labels: Array.from({length: 10}, (_, i) => (i - 5).toString()),
                datasets: [{
                    label: 'Count',
                    data: data.prs_dist,
                    backgroundColor: 'rgba(52, 152, 219, 0.6)'
                }]
            },
            options: { responsive: true, maintainAspectRatio: false }
        });

        // Risk Tier Donut
        new Chart(document.getElementById('tierChart'), {
            type: 'doughnut',
            data: {
                labels: Object.keys(data.risk_tiers),
                datasets: [{
                    data: Object.values(data.risk_tiers),
                    backgroundColor: ['#e74c3c', '#3498db', '#2ecc71']
                }]
            },
            options: { responsive: true, maintainAspectRatio: false }
        });

        // Model Performance
        const models = [...new Set(data.performance.map(d => d.Model))];
        new Chart(document.getElementById('modelChart'), {
            type: 'bar',
            data: {
                labels: models,
                datasets: [{
                    label: 'AUC',
                    data: models.map(m => data.performance.find(d => d.Model === m && d.Metric === 'AUC').Value),
                    backgroundColor: '#9b59b6'
                }]
            },
            options: { responsive: true, maintainAspectRatio: false }
        });

        // Correlation Bar
        new Chart(document.getElementById('corrChart'), {
            type: 'bar',
            data: {
                labels: data.correlations.map(d => d.Metric).slice(1, 6),
                datasets: [{
                    label: 'Correlation Coefficient',
                    data: data.correlations.map(d => d.Correlation).slice(1, 6),
                    backgroundColor: '#f1c40f'
                }]
            },
            options: { indexAxis: 'y', responsive: true, maintainAspectRatio: false }
        });
    </script>
</body>
</html>
""".replace('DATA_PLACEHOLDER', dashboard_json)

HTML(html_code)

HTML(html_code)
