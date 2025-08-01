```python
import csv
import os
from reportlab.lib.pagesizes import letter, A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle, PageBreak
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import inch, mm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from datetime import datetime

def csv_to_pdf(input_csv, output_pdf, title, intro_text, image_path):
    """
    将包含标题、查询条件和表格数据的CSV文件转换为精美的PDF报告
    
    参数:
    input_csv (str): 输入CSV文件路径
    output_pdf (str): 输出PDF文件路径
    title (str): PDF主标题
    intro_text (str): 介绍性文本
    image_path (str): 固定图片路径
    """
    
    # 创建PDF文档
    doc = SimpleDocTemplate(
        output_pdf,
        pagesize=A4,
        rightMargin=36,
        leftMargin=36,
        topMargin=72,
        bottomMargin=36
    )
    
    # 注册中文字体（确保系统有支持中文的字体）
    try:
        pdfmetrics.registerFont(TTFont('SimSun', 'simsun.ttc'))  # Windows系统
        pdfmetrics.registerFont(TTFont('STSong', 'STSong.ttf'))  # macOS系统
        font_name = 'SimSun'
    except:
        font_name = 'Helvetica'  # 回退到英文字体
    
    # 获取样式表
    styles = getSampleStyleSheet()
    
    # 自定义样式
    title_style = ParagraphStyle(
        'TitleStyle',
        parent=styles['Heading1'],
        fontName=font_name,
        fontSize=22,
        alignment=1,  # 居中
        spaceAfter=12,
        textColor=colors.HexColor('#2C3E50'),
        leading=28
    )
    
    subtitle_style = ParagraphStyle(
        'SubtitleStyle',
        parent=styles['Heading2'],
        fontName=font_name,
        fontSize=16,
        alignment=0,  # 左对齐
        spaceAfter=6,
        textColor=colors.HexColor('#2980B9'),
        leading=22
    )
    
    intro_style = ParagraphStyle(
        'IntroStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=12,
        spaceAfter=18,
        textColor=colors.HexColor('#34495E'),
        leading=18
    )
    
    condition_style = ParagraphStyle(
        'ConditionStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=11,
        spaceAfter=8,
        textColor=colors.HexColor('#7F8C8D'),
        leading=16
    )
    
    footer_style = ParagraphStyle(
        'FooterStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=9,
        textColor=colors.HexColor('#95A5A6'),
        alignment=2  # 右对齐
    )
    
    # 创建内容列表
    story = []
    
    # ==================== 封面页 ====================
    # 添加封面图片（全宽度）
    if os.path.exists(image_path):
        img = Image(image_path, width=6*inch, height=3*inch)
        img.hAlign = 'CENTER'
        story.append(img)
        story.append(Spacer(1, 0.5*inch))
    
    # 添加主标题
    story.append(Paragraph(title, title_style))
    story.append(Spacer(1, 0.5*inch))
    
    # 添加介绍文本
    intro_paragraphs = intro_text.split('\n')
    for para in intro_paragraphs:
        if para.strip():
            story.append(Paragraph(para.strip(), intro_style))
    
    # 添加报告信息
    story.append(Spacer(1, 1*inch))
    report_date = datetime.now().strftime("%Y年%m月%d日 %H:%M")
    story.append(Paragraph(f"报告生成时间: {report_date}", condition_style))
    story.append(Paragraph(f"数据来源: {os.path.basename(input_csv)}", condition_style))
    
    story.append(PageBreak())  # 封面页结束
    
    # ==================== 内容页 ====================
    # 读取CSV数据
    report_title = ""
    conditions = []
    header = []
    data = []
    
    with open(input_csv, 'r', encoding='utf-8') as csvfile:
        csv_reader = csv.reader(csvfile)
        
        # 读取报告标题（第一行）
        try:
            first_row = next(csv_reader)
            if first_row:
                report_title = first_row[0].strip('"')
        except StopIteration:
            pass
        
        # 读取查询条件（直到遇到空行或表头）
        for row in csv_reader:
            if not row:  # 跳过空行
                continue
            # 检查是否是表头（包含多个列）
            if len(row) > 1 and any(field.strip() for field in row):
                header = [field.strip('"') for field in row]
                break
            if row[0].strip():
                conditions.append(row[0].strip('"'))
        
        # 读取数据行
        for row in csv_reader:
            if any(field.strip() for field in row):  # 跳过空行
                cleaned_row = [field.strip('"') for field in row]
                data.append(cleaned_row)
    
    # 添加报告标题
    if report_title:
        story.append(Paragraph(report_title, subtitle_style))
        story.append(Spacer(1, 0.2*inch))
    
    # 添加查询条件
    if conditions:
        conditions_html = "<br/>".join([f"• {cond}" for cond in conditions])
        story.append(Paragraph(conditions_html, condition_style))
        story.append(Spacer(1, 0.3*inch))
    
    # 确保有数据
    if not header or not data:
        story.append(Paragraph("CSV文件中没有有效数据", intro_style))
        doc.build(story)
        return
    
    # 创建表格
    table_data = [header] + data
    
    # 自动计算列宽
    col_widths = [max(len(str(cell))*3.5 for cell in header]
    total_width = sum(col_widths)
    page_width = doc.width
    if total_width > page_width:
        ratio = page_width / total_width
        col_widths = [width * ratio for width in col_widths]
    
    table = Table(
        table_data, 
        colWidths=col_widths,
        repeatRows=1  # 重复表头
    )
    
    # 应用表格样式
    table_style = TableStyle([
        # 表头样式
        ('BACKGROUND', (0,0), (-1,0), colors.HexColor('#3498DB')),
        ('TEXTCOLOR', (0,0), (-1,0), colors.whitesmoke),
        ('FONTNAME', (0,0), (-1,0), font_name),
        ('FONTSIZE', (0,0), (-1,0), 12),
        ('BOLD', (0,0), (-1,0), 1),
        ('ALIGN', (0,0), (-1,0), 'CENTER'),
        ('VALIGN', (0,0), (-1,0), 'MIDDLE'),
        
        # 数据行样式
        ('FONTNAME', (0,1), (-1,-1), font_name),
        ('FONTSIZE', (0,1), (-1,-1), 10),
        ('TEXTCOLOR', (0,1), (-1,-1), colors.HexColor('#2C3E50')),
        ('BOTTOMPADDING', (0,0), (-1,-1), 8),
        ('TOPPADDING', (0,0), (-1,-1), 6),
        ('GRID', (0,0), (-1,-1), 0.5, colors.HexColor('#BDC3C7')),
        
        # 交替行颜色
        ('ROWBACKGROUNDS', (0,1), (-1,-1), 
         [colors.HexColor('#FFFFFF'), colors.HexColor('#F8F9F9')]),
        
        # 最后一行加粗
        ('FONTSIZE', (0,-1), (-1,-1), 11),
        ('BOLD', (0,-1), (-1,-1), 1),
        ('LINEABOVE', (0,-1), (-1,-1), 1, colors.HexColor('#E74C3C')),
    ])
    
    # 特殊列对齐（数字列右对齐）
    for i, col in enumerate(header):
        if any(col.lower().find(keyword) != -1 for keyword in ['amount', '金额', '数量', '总计']):
            table_style.add('ALIGN', (i,1), (i,-1), 'RIGHT')
    
    table.setStyle(table_style)
    story.append(table)
    story.append(Spacer(1, 0.3*inch))
    
    # 添加页脚
    footer_text = f"第 <page> 页，共 {len(data)} 条记录"
    story.append(Paragraph(footer_text, footer_style))
    
    # 生成PDF
    doc.build(story)

if __name__ == "__main__":
    # 示例用法
    intro_text = """
    本报告基于系统导出的原始交易数据生成，包含详细的交易记录和分析结果。
    报告数据已经过系统自动校验，确保准确性。如需进一步分析或对数据有疑问，
    请联系数据分析部门。
    """
    
    csv_to_pdf(
        input_csv='transaction_report.csv',  # 输入CSV文件
        output_pdf='交易报告.pdf',           # 输出PDF文件
        title='交易明细分析报告',            # PDF主标题
        intro_text=intro_text,               # 介绍文本
        image_path='company_banner.png'      # 封面图片路径
    )
```