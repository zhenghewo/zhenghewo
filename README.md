```python
import csv
import os
from reportlab.lib.pagesizes import letter, A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle, Frame, PageTemplate
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import inch, mm
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.enums import TA_LEFT, TA_CENTER, TA_RIGHT

def csv_to_pdf(input_csv, output_pdf, intro_text, image_path):
    """
    将CSV文件转换为带有固定布局的PDF报告
    
    参数:
    input_csv (str): 输入CSV文件路径
    output_pdf (str): 输出PDF文件路径
    intro_text (str): 介绍性文本
    image_path (str): 公司Logo路径
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
        # 尝试注册常见中文字体
        pdfmetrics.registerFont(TTFont('SimSun', 'simsun.ttc'))  # Windows系统
        pdfmetrics.registerFont(TTFont('STSong', 'STSong.ttf'))  # macOS系统
        font_name = 'SimSun'
    except:
        try:
            # 尝试其他常见中文字体
            pdfmetrics.registerFont(TTFont('SimHei', 'simhei.ttf'))
            font_name = 'SimHei'
        except:
            # 回退到英文字体
            font_name = 'Helvetica'
    
    # 获取样式表
    styles = getSampleStyleSheet()
    
    # 自定义样式
    title_style = ParagraphStyle(
        'TitleStyle',
        parent=styles['Heading1'],
        fontName=font_name,
        fontSize=24,
        alignment=TA_CENTER,
        spaceAfter=18,
        textColor=colors.HexColor('#2C3E50'),
        leading=28,
        fontWeight='bold'  # 加粗
    )
    
    subtitle_style = ParagraphStyle(
        'SubtitleStyle',
        parent=styles['Heading2'],
        fontName=font_name,
        fontSize=14,
        alignment=TA_LEFT,
        spaceAfter=6,
        textColor=colors.HexColor('#2980B9'),
        leading=20
    )
    
    intro_style = ParagraphStyle(
        'IntroStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=11,
        spaceAfter=12,
        textColor=colors.HexColor('#34495E'),
        leading=16,
        alignment=TA_LEFT
    )
    
    condition_title_style = ParagraphStyle(
        'ConditionTitleStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=11,
        textColor=colors.HexColor('#2C3E50'),
        leading=14,
        alignment=TA_LEFT,
        fontWeight='bold'  # 加粗
    )
    
    condition_value_style = ParagraphStyle(
        'ConditionValueStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=11,
        textColor=colors.HexColor('#7F8C8D'),
        leading=14,
        alignment=TA_LEFT
    )
    
    footer_style = ParagraphStyle(
        'FooterStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=9,
        textColor=colors.HexColor('#95A5A6'),
        alignment=TA_RIGHT
    )
    
    # 创建内容列表
    story = []
    
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
    
    # ==================== 页面布局 ====================
    
    # 添加标题（居中加粗大字号）
    if report_title:
        story.append(Paragraph(report_title, title_style))
        story.append(Spacer(1, 0.2*inch))
    
    # 创建顶部布局表格（2列：左侧为介绍文字，右侧为图片和查询条件）
    layout_table = []
    
    # 左侧：介绍文字
    intro_cell = [
        Paragraph(intro_text, intro_style)
    ]
    
    # 右侧：图片和查询条件
    right_cell = []
    
    # 添加公司Logo（右上角）
    if os.path.exists(image_path):
        img = Image(image_path, width=190, height=190*0.6)  # 宽度190点，高度按比例
        img.hAlign = 'RIGHT'
        right_cell.append(img)
        right_cell.append(Spacer(1, 0.1*inch))
    
    # 添加查询条件（图片下方）
    if conditions:
        # 添加"查询条件"标题
        right_cell.append(Paragraph("<b>查询条件</b>", condition_title_style))
        right_cell.append(Spacer(1, 0.05*inch))
        
        # 添加每个查询条件
        for cond in conditions:
            # 分割条件名和值
            if ':' in cond:
                parts = cond.split(':', 1)
                title = parts[0].strip() + ":"
                value = parts[1].strip()
            else:
                title = cond
                value = ""
            
            # 创建条件行（标题加粗）
            cond_row = [
                Paragraph(f"<b>{title}</b>", condition_title_style),
                Paragraph(value, condition_value_style)
            ]
            right_cell.append(cond_row)
            right_cell.append(Spacer(1, 0.05*inch))
    
    # 构建布局表格（2列）
    layout_table = Table([
        [intro_cell, right_cell]
    ], colWidths=[doc.width * 0.65, doc.width * 0.35])
    
    # 应用无边框样式
    layout_table.setStyle(TableStyle([
        ('VALIGN', (0,0), (-1,-1), 'TOP'),
        ('LEFTPADDING', (0,0), (-1,-1), 0),
        ('RIGHTPADDING', (0,0), (-1,-1), 0),
        ('BOTTOMPADDING', (0,0), (-1,-1), 0),
        ('TOPPADDING', (0,0), (-1,-1), 0),
        ('ALIGN', (0,0), (0,0), 'LEFT'),
        ('ALIGN', (1,0), (1,0), 'RIGHT'),
        ('BACKGROUND', (0,0), (-1,-1), colors.white),
    ]))
    
    story.append(layout_table)
    story.append(Spacer(1, 0.3*inch))
    
    # ==================== 数据表格 ====================
    
    # 确保有数据
    if not header or not data:
        story.append(Paragraph("CSV文件中没有有效数据", intro_style))
        doc.build(story)
        return
    
    # 创建表格
    table_data = [header] + data
    
    # 自动计算列宽（根据内容长度）
    col_widths = [max(len(str(cell)) * 6 for cell in col) for col in zip(*table_data)]
    total_width = sum(col_widths)
    page_width = doc.width
    
    # 如果总宽度超过页面宽度，按比例缩小
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
        
        # 最后一行特殊样式
        ('FONTSIZE', (0,-1), (-1,-1), 11),
        ('LINEBELOW', (0,-1), (-1,-1), 1, colors.HexColor('#E74C3C')),
    ])
    
    # 特殊列对齐（数字列右对齐）
    for i, col_name in enumerate(header):
        col_name_lower = col_name.lower()
        if any(keyword in col_name_lower for keyword in ['amount', '金额', '数量', '总计', '价格', '费用', 'total', 'price']):
            table_style.add('ALIGN', (i,1), (i,-1), 'RIGHT')
    
    table.setStyle(table_style)
    story.append(table)
    story.append(Spacer(1, 0.2*inch))
    
    # 添加页脚
    footer_text = f"生成时间: {os.path.basename(input_csv)} | 共 {len(data)} 条记录"
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
        intro_text=intro_text,               # 介绍文本
        image_path='company_logo.png'        # 公司Logo路径
    )
```