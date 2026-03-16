import React, { useState, useEffect } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import api from '../utils/api';
import { Printer, ArrowLeft } from 'lucide-react';
import logo from '../assets/logo/Ram2Logo.png';

interface TransactionDetail {
    id: number;
    itemId: number;
    sentQty: number;
    item: {
        name: string;
        isUniform: boolean;
        itemSize?: string;
        itemType?: string;
        department?: { name: string };
    };
    isInfected: boolean;
}

interface Transaction {
    id: number;
    docNo: string;
    type: string;
    date: string;
    shift: string;
    isInfected: boolean;
    dirtyWeightMorning: number;
    dirtyWeightAfternoon: number;
    cleanWeightMorning?: number;
    cleanWeightAfternoon?: number;
    linenStatus?: string;
    details: TransactionDetail[];
    hospitalStaffSender: { displayname: string };
    company?: { name: string };
    externalStaffName?: string;
    department?: { name: string };
    deptStaffName?: string;
    parent?: { docNo: string };
}

const TransactionReceipt: React.FC = () => {
    const { id } = useParams<{ id: string }>();
    const [tx, setTx] = useState<Transaction | null>(null);
    const [loading, setLoading] = useState(true);
    const navigate = useNavigate();

    useEffect(() => {
        const fetchTransaction = async () => {
            try {
                const res = await api.get('/transactions');
                const found = res.data.find((t: any) => t.id === Number(id));
                setTx(found);

                // Calculate pending balances per itemId
                const pending: Record<number, number> = {};
                res.data.forEach((t: any) => {
                    t.details?.forEach((d: any) => {
                        const itemId = d.itemId;
                        const diff = (d.sentQty || 0) - (d.receivedQty || 0);
                        if (diff > 0) {
                            pending[itemId] = (pending[itemId] || 0) + diff;
                        }
                    });
                });
            } catch (err) {
                console.error('Error fetching transaction');
            } finally {
                setLoading(false);
            }
        };
        fetchTransaction();
    }, [id]);

    useEffect(() => {
        document.body.classList.add('print-receipt-mode');
        return () => {
            document.body.classList.remove('print-receipt-mode');
        };
    }, []);

    const handlePrint = () => {
        window.print();
    };

    if (loading) return <div className="p-10 text-center">กำลังโหลดข้อมูล...</div>;
    if (!tx) return <div className="p-10 text-center text-red-500">ไม่พบข้อมูลเอกสาร</div>;

    // Helper to chunk items (20 items per page)
    const itemsPerPage = 15;
    const itemChunks = [];
    for (let i = 0; i < tx.details.length; i += itemsPerPage) {
        itemChunks.push(tx.details.slice(i, i + itemsPerPage));
    }

    // If no items, show at least one empty page
    if (itemChunks.length === 0) itemChunks.push([]);

    return (
        <div className="min-h-screen bg-gray-100 p-2 sm:p-4 md:p-8 flex flex-col items-center gap-6 sm:gap-10 print:gap-0 print:p-0">
            {/* Action Bar (Hidden when printing) */}
            <div className="w-full max-w-[210mm] flex justify-between items-center print:hidden">
                <button onClick={() => navigate(-1)} className="flex items-center gap-2 text-gray-600 hover:text-black transition font-bold">
                    <ArrowLeft size={20} /> ย้อนกลับ
                </button>
                <button onClick={handlePrint} className="bg-blue-600 text-white px-6 py-2 rounded-lg flex items-center gap-2 shadow-lg hover:bg-blue-700 transition font-bold">
                    <Printer size={20} /> พิมพ์เอกสาร (Print)
                </button>
            </div>

            {/* Render each chunk as a separate page */}
            {itemChunks.map((chunk, chunkIndex) => (
                <div
                    key={chunkIndex}
                    className="bg-white w-full max-w-[210mm] min-h-[max(282mm,100vh)] sm:min-h-[282mm] p-[6mm] sm:p-[15mm] shadow-2xl print:shadow-none print:p-[12mm] flex flex-col receipt-page-container box-border relative overflow-hidden"
                    style={{
                        breakAfter: chunkIndex === itemChunks.length - 1 ? 'auto' : 'always',
                        pageBreakAfter: chunkIndex === itemChunks.length - 1 ? 'auto' : 'always'
                    }}
                >
                    {/* Header (Full on page 1, Simple on others) */}
                    <div className="flex justify-between items-start border-b-2 border-black pb-4 mb-6">
                        <div className="flex items-center gap-4">
                            <img src={logo} alt="Ram2 Logo" className="w-24 h-24 object-contain" />
                            <div>
                                <h1 className="text-lg font-black mb-0.5 text-gray-900 leading-none">
                                    {chunkIndex === 0 ? 'ใบนำส่งผ้าเพื่อซัก' : 'ใบนำส่งผ้าเพื่อซัก (ต่อ)'}
                                </h1>
                                <p className="text-[11px] font-bold text-slate-800">Linen Laundry Export Document</p>
                            </div>
                        </div>
                        <div className="text-right">
                            <div className="text-xs sm:text-sm font-black text-slate-900 leading-tight">เลขที่เอกสาร: {tx.docNo}</div>
                            <div className="text-[8px] sm:text-[9px] font-bold text-slate-600">วันที่: {new Date(tx.date).toLocaleDateString('th-TH')}</div>
                        </div>
                    </div>

                    {/* Info Grid (Only on Page 1) */}
                    {chunkIndex === 0 && (
                        <div className="grid grid-cols-2 gap-6 mb-6 text-xs">
                            <div className="space-y-1 border-l-[3px] border-slate-100 pl-3">
                                <p><span className="text-slate-900 font-bold text-[9px] uppercase block tracking-wider">ประเภทงาน (Type)</span> <span className="font-black text-slate-800">{tx.type === 'RE' ? 'ส่งซักซ้ำ (Rewash)' : (tx.type === 'OUT' ? 'ส่งซักภายนอก (External)' : 'ส่งซักภายใน (Internal)')}</span></p>
                                <p><span className="text-slate-900 font-bold text-[9px] uppercase block tracking-wider">หน่วยงาน/บริษัท (Source)</span> <span className="font-black text-slate-800">{(tx.type === 'OUT' || tx.type === 'RE') ? tx.company?.name : tx.department?.name}</span></p>
                                {tx.parent && (
                                    <p className="mt-2 p-1.5 border-2 border-slate-900 text-slate-900 rounded-md inline-block">
                                        <span className="text-[8px] font-black uppercase tracking-widest block">ซักซ้ำจากเลขที่ (Rewash from)</span>
                                        <span className="font-black text-[11px]">{tx.parent.docNo}</span>
                                    </p>
                                )}
                            </div>
                            <div className="space-y-1 border-l-[3px] border-slate-100 pl-3">
                                <p>
                                    <span className="text-slate-900 font-bold text-[9px] uppercase block tracking-wider">สถานะ (Status)</span>
                                    {tx.isInfected
                                        ? <span className="font-black text-slate-900"> [!] ติดเชื้อ (Infected) [!]</span>
                                        : (tx.linenStatus === 'CLEAN'
                                            ? <span className="font-black text-emerald-600">ผ้าสะอาด (Clean)</span>
                                            : <span className="font-black text-rose-600">ผ้าสกปรก (Dirty)</span>)
                                    }
                                </p>
                                {tx.type !== 'RE' && (
                                    <p>
                                        <span className="text-slate-900 font-bold text-[9px] uppercase block tracking-wider">น้ำหนักรวม (Weight)</span>
                                        <span className="font-black text-slate-800">
                                            {tx.linenStatus === 'CLEAN'
                                                ? `${(tx.cleanWeightMorning || 0) + (tx.cleanWeightAfternoon || 0)} กก.`
                                                : `${tx.dirtyWeightMorning + tx.dirtyWeightAfternoon} กก.`
                                            }
                                        </span>
                                        <span className="text-[9px] text-slate-900 font-bold italic ml-1">
                                            {tx.linenStatus === 'CLEAN'
                                                ? `(รอบเช้า: ${tx.cleanWeightMorning || 0} / รอบบ่าย: ${tx.cleanWeightAfternoon || 0})`
                                                : `(รอบเช้า: ${tx.dirtyWeightMorning} / รอบบ่าย: ${tx.dirtyWeightAfternoon})`
                                            }
                                        </span>
                                    </p>
                                )}
                            </div>
                        </div>
                    )}

                    {/* Items Table for this chunk */}
                    <div className="rounded-xl border-[1px] sm:border-2 border-slate-200 overflow-hidden mb-4 overflow-x-auto">
                        <table className="w-full border-collapse min-w-[600px] sm:min-w-0">
                            <thead>
                                <tr className="bg-slate-50 border-b-2 border-slate-200">
                                    <th className="p-2 text-center w-10 text-[9px] font-black text-slate-900 uppercase tracking-wider">#</th>
                                    <th className="p-2 text-left text-[9px] font-black text-slate-900 uppercase tracking-wider">รายการผ้า (Linen Items)</th>
                                    <th className="p-2 text-center w-18 text-[9px] font-black text-slate-900 uppercase tracking-wider">ประเภท</th>
                                    <th className="p-2 text-center w-12 text-[9px] font-black text-slate-900 uppercase tracking-wider">Size</th>
                                    <th className="p-2 text-center w-24 text-[9px] font-black text-slate-900 uppercase tracking-wider">แผนก (Dept.)</th>
                                    <th className="p-2 text-center w-20 text-[9px] font-black text-slate-900 uppercase tracking-wider">จำนวนคีย์</th>
                                    <th className="p-2 text-center w-14 text-[9px] font-black text-slate-900 uppercase tracking-wider">ค้าง</th>

                                </tr>
                            </thead>
                            <tbody className="divide-y divide-slate-100 italic font-medium text-slate-700">
                                {chunk.map((d, index) => (
                                    <tr key={d.id} className="hover:bg-slate-50/50 transition-colors">
                                        <td className="p-2 text-center text-slate-900 font-bold text-[9px]">{chunkIndex * itemsPerPage + index + 1}</td>
                                        <td className="p-2 font-bold text-slate-900 text-[10px]">
                                            {d.item.name}
                                            {d.item.isUniform && <span className="ml-1 text-[7px] bg-slate-100 text-black px-1 py-0.5 rounded-full uppercase tracking-tighter border border-slate-200">Uniform</span>}
                                            {d.isInfected && <span className="ml-1 text-[8px] text-black px-2 py-0.5 rounded-md uppercase tracking-tighter font-black border-2 border-black bg-white shadow-none">*** ติดเชื้อ ***</span>}
                                        </td>
                                        <td className="p-2 text-center text-[8px] font-bold text-slate-900 uppercase">
                                            {(() => {
                                                const mapping: Record<string, string> = {
                                                    'SHIRT': 'เสื้อ', 'PANTS': 'กางเกง', 'DOCTOR_SUIT': 'เสื้อสูทแพทย์',
                                                    'DRESS': 'ชุดเดรส/กระโปรง', 'SET': 'ชุดเข้าเซ็ท', 'OTHER': 'อื่นๆ'
                                                };
                                                return d.item.itemType ? (mapping[d.item.itemType.toUpperCase()] || d.item.itemType) : '-';
                                            })()}
                                        </td>
                                        <td className="p-2 text-center text-[9px] font-bold text-slate-600">{d.item.itemSize || '-'}</td>
                                        <td className="p-2 text-center text-[8px] font-bold text-slate-900 uppercase tracking-tighter">
                                            {d.item.department?.name || tx.department?.name || '-'}
                                        </td>

                                        <td className="p-2 text-center font-black text-blue-600 text-[10px]">
                                            {d.sentQty.toLocaleString()}
                                            <span className="ml-1 text-[7px] text-slate-400 font-bold uppercase">
                                                {d.item.itemType?.toUpperCase() === 'DOCTOR_SUIT' ? 'ตัว' : 'ชิ้น'}
                                            </span>
                                        </td>
                                        <td className="p-2 text-center font-black text-red-600 text-[10px]">
                                            {/* {outstandingBalances[d.itemId] || 0} */}
                                        </td>
                                    </tr>
                                ))}
                                {/* Fill remaining space if last page and small amount of items */}
                                {chunkIndex === itemChunks.length - 1 && chunk.length < 5 && Array.from({ length: 5 - chunk.length }).map((_, i) => (
                                    <tr key={`empty-${i}`}>
                                        <td className="p-3 h-10"></td>
                                        <td className="p-3 border-r border-slate-50"></td>
                                        <td className="p-3 border-r border-slate-50"></td>
                                        <td className="p-3 border-r border-slate-50"></td>
                                        <td className="p-3 border-r border-slate-50"></td>
                                        <td className="p-3 border-r border-slate-50 text-red-100"></td>
                                        <td className="p-3 border-r border-slate-50 text-blue-100"></td>
                                    </tr>
                                ))}
                            </tbody>
                            {/* Grand Total Footer (Only on Last Page) */}
                            {chunkIndex === itemChunks.length - 1 && (
                                <tfoot className="bg-slate-900 text-white">
                                    <tr className="font-black">
                                        <td colSpan={6} className="p-3 text-right text-[9px] uppercase tracking-widest text-white">รวมจำนวนคีย์ทั้งหมด (Grand Total)</td>
                                        <td className="p-3 text-center text-[13px]">
                                            {tx.details.reduce((acc, curr) => acc + curr.sentQty, 0).toLocaleString()} 
                                            <span className="text-[7px] text-white font-bold ml-1 uppercase">
                                                {(() => {
                                                    const allDoctorSuits = tx.details.every(d => d.item.itemType?.toUpperCase() === 'DOCTOR_SUIT');
                                                    return allDoctorSuits ? 'ตัว' : 'ชิ้น';
                                                })()}
                                            </span>
                                        </td>
                                    </tr>
                                </tfoot>
                            )}
                        </table>
                    </div>

                    {/* Signatures Area - Pushed to bottom (Only on Last Page) */}
                    {chunkIndex === itemChunks.length - 1 && (
                        <>
                            <div className="mt-auto grid grid-cols-2 gap-12 pt-4">
                                <div className="p-4 text-center space-y-8">
                                    <p className="font-black text-slate-900 uppercase text-[10px] tracking-[0.2em] mb-4">
                                        {tx.type === 'IN' ? 'เจ้าหน้าที่หน่วยงาน (Dept. Sender)' : 'ฝ่ายโรงพยาบาล (Hospital Sender)'}
                                    </p>
                                    <div className="border-b-2 border-slate-200 w-3/4 mx-auto pt-6"></div>
                                    <div className="space-y-1">
                                        <p className="font-black text-slate-900 text-xs">
                                            ({tx.type === 'IN' ? tx.deptStaffName : tx.hospitalStaffSender.displayname})
                                        </p>
                                        <p className="text-[9px] font-bold text-slate-600 uppercase tracking-wider">
                                            {tx.type === 'IN' ? 'ผู้มอบผ้า (Nurse/Staff)' : 'ผู้ส่งมอบผ้า'}
                                        </p>
                                    </div>
                                </div>
                                <div className="p-4 text-center space-y-8 border-l border-slate-100">
                                    <p className="font-black text-slate-900 uppercase text-[10px] tracking-[0.2em] mb-4">
                                        {tx.type === 'IN' ? 'ฝ่ายโรงพยาบาล (Laundry Receiver)' : 'ฝ่ายผู้รับผ้า (External Receiver)'}
                                    </p>
                                    <div className="border-b-2 w-3/4 mx-auto pt-6 border-slate-200"></div>
                                    <div className="space-y-1">
                                        <p className="font-black text-slate-900 text-xs">
                                            ({tx.type === 'IN' ? tx.hospitalStaffSender.displayname : (tx.externalStaffName || '-')})
                                        </p>
                                        <p className="text-[9px] font-bold text-slate-600 uppercase tracking-wider">
                                            {tx.type === 'IN' ? 'เจ้าหน้าที่ห้องผ้า (ผู้รับมอบ)' : 'พนักงานบริษัท / ผู้รับ'}
                                        </p>
                                    </div>
                                </div>
                            </div>
                        </>
                    )}


                    {/* Centered Page Numbering and Bottom-Right Document Code */}
                    <div className="mt-auto pt-4 flex justify-between items-end">
                        <div className="w-1/3"></div> {/* Spacer to balance flexbox */}

                        <div className="w-1/3 text-center">
                            {itemChunks.length > 1 && (
                                <div className="text-[10px] font-black text-slate-900 uppercase tracking-widest">
                                    หน้า {chunkIndex + 1} / {itemChunks.length}
                                </div>
                            )}
                        </div>

                        <div className="w-1/3 text-right">
                            <div className="text-[9px] font-black text-slate-900 uppercase tracking-tighter">
                                {tx.type === 'IN' ? 'FLND-0001-01' : 'FLND-0002-01'}
                            </div>
                        </div>
                    </div>
                </div>
            ))}

            {/* CSS for print mode */}
            <style dangerouslySetInnerHTML={{
                __html: `
                @media print {
                    @page { 
                        size: A4; 
                        margin: 0 !important; 
                    }
                    * { 
                        box-sizing: border-box !important; 
                    }
                    html, body {
                        height: auto !important;
                        margin: 0 !important;
                        padding: 0 !important;
                        background: white !important;
                        overflow: visible !important;
                    }
                    body.print-receipt-mode {
                        display: block !important;
                        background: white !important;
                    }
                    body.print-receipt-mode .min-h-screen {
                        display: block !important;
                        height: auto !important;
                        min-height: 0 !important;
                        padding: 0 !important;
                        gap: 0 !important;
                    }
                    body.print-receipt-mode .receipt-page-container {
                        margin: 0 auto !important;
                        box-shadow: none !important;
                        border: none !important;
                        overflow: hidden !important;
                        display: flex !important;
                        flex-direction: column !important;
                        width: 210mm !important;
                        height: 282mm !important;
                    }
                    .print\\:hidden { display: none !important; }
                }
            `}} />
        </div>
    );
};

export default TransactionReceipt;
