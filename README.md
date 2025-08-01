```python
import csv
import os
from datetime import datetime
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image, Table, TableStyle, Frame, PageTemplate
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib import colors
from reportlab.lib.units import mm, inch
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.enums import TA_LEFT, TA_CENTER, TA_RIGHT

def csv_to_pdf(input_csv, output_pdf, ecochash_statement, intro_text, image_path):
    """
    将CSV文件转换为带有精确布局的PDF报告
    
    参数:
    input_csv (str): 输入CSV文件路径
    output_pdf (str): 输出PDF文件路径
    ecochash_statement (str): 左上角的标题文本
    intro_text (str): 介绍性文本
    image_path (str): 公司Logo路径
    """
    
    # 创建PDF文档 (宽度1900点)
    custom_size = (1900, 2800)  # 自定义页面大小
    doc = SimpleDocTemplate(
        output_pdf,
        pagesize=custom_size,
        rightMargin=36,
        leftMargin=36,
        topMargin=36,
        bottomMargin=36
    )
    
    # 注册中文字体
    try:
        pdfmetrics.registerFont(TTFont('SimSun', 'simsun.ttc'))
        font_name = 'SimSun'
    except:
        font_name = 'Helvetica'
    
    # 获取样式表
    styles = getSampleStyleSheet()
    
    # 自定义样式
    title_style = ParagraphStyle(
        'TitleStyle',
        parent=styles['Heading1'],
        fontName=font_name,
        fontSize=36,
        alignment=TA_LEFT,
        spaceAfter=12,
        textColor=colors.HexColor('#2C3E50'),
        leading=40,
        fontWeight='bold'
    )
    
    intro_style = ParagraphStyle(
        'IntroStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=20,
        spaceAfter=12,
        textColor=colors.HexColor('#34495E'),
        leading=26,
        alignment=TA_LEFT
    )
    
    condition_title_style = ParagraphStyle(
        'ConditionTitleStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=22,
        textColor=colors.HexColor('#2C3E50'),
        leading=26,
        alignment=TA_LEFT,
        fontWeight='bold'
    )
    
    condition_value_style = ParagraphStyle(
        'ConditionValueStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=22,
        textColor=colors.HexColor('#7F8C8D'),
        leading=26,
        alignment=TA_LEFT
    )
    
    footer_style = ParagraphStyle(
        'FooterStyle',
        parent=styles['BodyText'],
        fontName=font_name,
        fontSize=18,
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
        
        # 读取查询条件
        for row in csv_reader:
            if not row:
                continue
            if len(row) > 1 and any(field.strip() for field in row):
                header = [field.strip('"') for field in row]
                break
            if row[0].strip():
                conditions.append(row[0].strip('"'))
    
        # 读取数据行
        for row in csv_reader:
            if any(field.strip() for field in row):
                cleaned_row = [field.strip('"') for field in row]
                data.append(cleaned_row)
    
    # ==================== 顶部布局 ====================
    # 创建顶部布局表格（2列：左上角标题 + 右上角图片）
    top_table_data = [
        [
            Paragraph(ecochash_statement, title_style),
            None  # 图片将在下方单独添加
        ]
    ]
    
    top_table = Table(top_table_data, colWidths=[doc.width * 0.7, doc.width * 0.3])
    
    # 应用顶部表格样式
    top_table.setStyle(TableStyle([
        ('VALIGN', (0,0), (-1,-1), 'TOP'),
        ('LEFTPADDING', (0,0), (0,0), 0),
        ('RIGHTPADDING', (0,0), (0,0), 0),
        ('BOTTOMPADDING', (0,0), (-1,-1), 0),
        ('TOPPADDING', (0,0), (-1,-1), 0),
        ('BACKGROUND', (0,0), (-1,-1), colors.white),
    ]))
    
    story.append(top_table)
    
    # 添加右上角图片
    if os.path.exists(image_path):
        img = Image(image_path, width=190, height=190*0.8)
        img.hAlign = 'RIGHT'
        story.append(img)
    
    story.append(Spacer(1, 0.5*inch))
    
    # ==================== 中部布局 ====================
    # 创建中部布局表格（2列：左侧介绍文字 + 右侧查询条件）
    middle_table_data = []
    
    # 左侧：介绍文字
    left_cell = [
        Paragraph(intro_text, intro_style)
    ]
    
    # 右侧：打印日期和查询条件
    right_cell = []
    
    # 添加打印日期
    print_date = datetime.now().strftime("%Y年%m月%d日 %H:%M")
    right_cell.append(Paragraph(f"<b>打印日期:</b> {print_date}", condition_value_style))
    right_cell.append(Spacer(1, 0.2*inch))
    
    # 添加查询条件标题
    right_cell.append(Paragraph("<b>查询条件</b>", condition_title_style))
    right_cell.append(Spacer(1, 0.1*inch))
    
    # 添加每个查询条件
    if conditions:
        for cond in conditions:
            if ':' in cond:
                parts = cond.split(':', 1)
                title = parts[0].strip()
                value = parts[1].strip()
                cond_text = f"<b>{title}:</b> {value}"
            else:
                cond_text = f"<b>{cond}</b>"
            
            right_cell.append(Paragraph(cond_text, condition_value_style))
            right_cell.append(Spacer(1, 0.05*inch))
    
    # 构建中部表格（2列）
    middle_table = Table([
        [left_cell, right_cell]
    ], colWidths=[doc.width * 0.6, doc.width * 0.4])
    
    # 应用中部表格样式
    middle_table.setStyle(TableStyle([
        ('VALIGN', (0,0), (-1,-1), 'TOP'),
        ('LEFTPADDING', (0,0), (-1,-1), 0),
        ('RIGHTPADDING', (0,0), (-1,-1), 0),
        ('BOTTOMPADDING', (0,0), (-1,-1), 0),
        ('TOPPADDING', (0,0), (-1,-1), 0),
        ('ALIGN', (0,0), (0,0), 'LEFT'),
        ('ALIGN', (1,0), (1,0), 'LEFT'),
        ('BACKGROUND', (0,0), (-1,-1), colors.white),
    ]))
    
    story.append(middle_table)
    story.append(Spacer(1, 0.5*inch))
    
    # ==================== 数据表格 ====================
    if header and data:
        # 创建表格数据（表头 + 数据）
        table_data = [header] + data
        
        # 创建表格
        table = Table(
            table_data, 
            repeatRows=1,
            colWidths=[doc.width / len(header)] * len(header)
        )
        
        # 应用表格样式
        table_style = TableStyle([
            # 表头样式
            ('BACKGROUND', (0,0), (-1,0), colors.HexColor('#3498DB')),
            ('TEXTCOLOR', (0,0), (-1,0), colors.whitesmoke),
            ('FONTNAME', (0,0), (-1,0), font_name),
            ('FONTSIZE', (0,0), (-1,0), 26),
            ('BOLD', (0,0), (-1,0), 1),
            ('ALIGN', (0,0), (-1,0), 'CENTER'),
            ('VALIGN', (0,0), (-1,0), 'MIDDLE'),
            
            # 数据行样式
            ('FONTNAME', (0,1), (-1,-1), font_name),
            ('FONTSIZE', (0,1), (-1,-1), 22),
            ('TEXTCOLOR', (0,1), (-1,-1), colors.HexColor('#2C3E50')),
            ('BOTTOMPADDING', (0,0), (-1,-1), 12),
            ('TOPPADDING', (0,0), (-1,-1), 10),
            ('GRID', (0,0), (-1,-1), 1, colors.HexColor('#BDC3C7')),
            
            # 交替行颜色
            ('ROWBACKGROUNDS', (0,1), (-1,-1), 
             [colors.HexColor('#FFFFFF'), colors.HexColor('#F8F9F9')]),
        ])
        
        # 特殊列对齐
        for i, col_name in enumerate(header):
            col_name_lower = col_name.lower()
            if any(keyword in col_name_lower for keyword in 
                   ['amount', '金额', '数量', '总计', '价格', '费用', 'total', 'price']):
                table_style.add('ALIGN', (i,1), (i,-1), 'RIGHT')
        
        table.setStyle(table_style)
        story.append(table)
        story.append(Spacer(1, 0.3*inch))
        
        # 添加页脚
        footer_text = f"共 {len(data)} 条记录 | 生成时间: {print_date}"
        story.append(Paragraph(footer_text, footer_style))
    else:
        story.append(Paragraph("CSV文件中没有有效数据", intro_style))
    
    # 生成PDF
    doc.build(story)

if __name__ == "__main__":
    # 示例用法
    ecochash_statement = "ECOCASH 交易明细报表"
    intro_text = """
    本报表包含系统所有交易明细记录，数据来源于核心交易数据库。
    报表内容已通过财务系统验证，确保准确性。如有疑问，请联系财务部门。
    """
    
    csv_to_pdf(
        input_csv='transaction_report.csv',  # 输入CSV文件
        output_pdf='交易报告.pdf',           # 输出PDF文件
        ecochash_statement=ecochash_statement,  # 左上角标题
        intro_text=intro_text,               # 介绍文本
        image_path='ecocash.png'             # 公司Logo路径
    )
```