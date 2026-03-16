import React, { useState, useEffect } from 'react';
import { useParams, useNavigate, useSearchParams } from 'react-router-dom';
import api from '../utils/api';
import SearchableSelect from '../components/SearchableSelect';
import { Plus, Trash2, Save, X, ArrowLeft, Activity, ClipboardList, Copy } from 'lucide-react';

interface LinenItem {
    id: number;
    name: string;
    category: string;
    isUniform: boolean;
    itemSize?: string;
    itemType?: string;
    department?: { name: string };
}

interface Company { id: number; name: string }
interface Department { id: number; name: string }

const TransactionForm: React.FC = () => {
    const { type, id } = useParams<{ type: string; id: string }>(); // 'in', 'out' or just id for edit
    const isEdit = !!id;
    const [searchParams] = useSearchParams();
    const parentId = searchParams.get('parentId');
    const isRewash = !!parentId;

    const isOut = type?.toLowerCase() === 'out' || isRewash; // Rewash defaults to similar UI as OUT
    const [headerIsOut, setHeaderIsOut] = useState(isOut);
    const [docNo, setDocNo] = useState('');
    const navigate = useNavigate();

    const [linenItems, setLinenItems] = useState<LinenItem[]>([]);
    const [companies, setCompanies] = useState<Company[]>([]);
    const [departments, setDepartments] = useState<Department[]>([]);

    // Header State
    const [shift, setShift] = useState('MORNING');
    const [isInfected, setIsInfected] = useState(false);
    const [dirtyWeightMorning, setDirtyWeightMorning] = useState(0);
    const [dirtyWeightAfternoon, setDirtyWeightAfternoon] = useState(0);
    const [cleanWeightMorning, setCleanWeightMorning] = useState(0);
    const [cleanWeightAfternoon, setCleanWeightAfternoon] = useState(0);
    const [weightStatus, setWeightStatus] = useState<'DIRTY' | 'CLEAN'>('DIRTY');
    const [companyId, setCompanyId] = useState('');
    const [externalStaffName, setExternalStaffName] = useState('');
    const [departmentId, setDepartmentId] = useState('');
    const [deptStaffName, setDeptStaffName] = useState('');

    // Items State
    const [selectedItems, setSelectedItems] = useState<{ id?: number; itemId: string; sentQty: number; isInfected: boolean; receivedQty?: number; remark?: string }[]>([
        { itemId: '', sentQty: 1, isInfected: false, receivedQty: 0, remark: '' }
    ]);

    const [loading, setLoading] = useState(false);
    const [error, setError] = useState('');

    useEffect(() => {
        const fetchData = async () => {
            try {
                const [itemsRes, compRes, deptRes, txRes] = await Promise.all([
                    api.get('/linen-items'),
                    api.get('/meta/companies'),
                    api.get('/meta/departments'),
                    api.get('/transactions')
                ]);
                setLinenItems(itemsRes.data);
                setCompanies(compRes.data);
                setDepartments(deptRes.data);

                // If editing, fetch transaction data
                if (isEdit) {
                    await api.get(`/transactions`); // Fetch all if needed or specific ID
                    // Note: Ideally backend should have GET /transactions/:id
                    // Let's check if the list has what we need or find it
                    const currentTx = txRes.data.find((t: any) => t.id === Number(id));
                    if (currentTx) {
                        setShift(currentTx.shift);
                        setIsInfected(currentTx.isInfected);
                        setDirtyWeightMorning(currentTx.dirtyWeightMorning);
                        setDirtyWeightAfternoon(currentTx.dirtyWeightAfternoon);
                        setCleanWeightMorning(currentTx.cleanWeightMorning || 0);
                        setCleanWeightAfternoon(currentTx.cleanWeightAfternoon || 0);
                        setCompanyId(currentTx.companyId || '');
                        setExternalStaffName(currentTx.externalStaffName || '');
                        setDepartmentId(currentTx.departmentId || '');
                        setDeptStaffName(currentTx.deptStaffName || '');
                        setHeaderIsOut(currentTx.type === 'OUT');
                        setDocNo(currentTx.docNo);

                        if (currentTx.details && currentTx.details.length > 0) {
                            setSelectedItems(currentTx.details.map((d: any) => ({
                                id: d.id,
                                itemId: String(d.itemId),
                                sentQty: d.sentQty,
                                isInfected: !!d.isInfected,
                                receivedQty: d.receivedQty || 0,
                                remark: d.remark || ''
                            })));
                        }
                    }
                }

                // If Rewash, fetch parent data
                if (parentId && !isEdit) {
                    const parentTx = txRes.data.find((t: any) => t.id === Number(parentId));
                    if (parentTx) {
                        setCompanyId(parentTx.companyId || '');
                        setExternalStaffName(parentTx.externalStaffName || '');
                        setDepartmentId(parentTx.departmentId || '');
                        setDeptStaffName(parentTx.deptStaffName || '');
                        setHeaderIsOut(parentTx.type === 'OUT');
                        setDocNo('RE-AUTO'); // Placeholder until saved

                        if (parentTx.details && parentTx.details.length > 0) {
                            // If transaction is COMPLETED, all items were received, so we rewash the full original amount.
                            // If it's PENDING, we rewash only the remaining (outstanding) amount.
                            const isParentCompleted = parentTx.status === 'COMPLETED';

                            const rewashItems = parentTx.details
                                .map((d: any) => {
                                    const remaining = d.sentQty - d.receivedQty;
                                    return {
                                        itemId: String(d.itemId),
                                        sentQty: isParentCompleted ? d.sentQty : (remaining > 0 ? remaining : d.sentQty),
                                        isInfected: !!d.isInfected,
                                        receivedQty: 0,
                                        remark: `ซักซ้ำจาก ${parentTx.docNo}`
                                    };
                                })
                                .filter((item: any) => item.sentQty > 0);

                            if (rewashItems.length > 0) {
                                setSelectedItems(rewashItems);
                            }
                        }
                    }
                }
            } catch (err) {
                console.error('Failed to fetch initial data');
            }
        };
        fetchData();
    }, [isEdit, id]);

    const addItemRow = () => setSelectedItems([...selectedItems, { itemId: '', sentQty: 1, isInfected: false, receivedQty: 0, remark: '' }]);
    const removeItemRow = (index: number) => {
        const newItems = [...selectedItems];
        newItems.splice(index, 1);
        setSelectedItems(newItems);
    };

    const updateItem = (index: number, field: string, value: any) => {
        const newItems = [...selectedItems];
        (newItems[index] as any)[field] = value;
        setSelectedItems(newItems);
    };

    const duplicateItem = (index: number) => {
        const { id, ...itemToCopy } = selectedItems[index];
        const newItems = [...selectedItems];
        newItems.splice(index + 1, 0, { ...itemToCopy, id: undefined }); // Explicitly reset ID
        setSelectedItems(newItems);
    };

    const totalItems = selectedItems.filter(i => i.itemId).length;
    const totalQty = selectedItems.reduce((sum, i) => sum + (Number(i.sentQty) || 0), 0);

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        if (selectedItems.some(i => !i.itemId || i.sentQty <= 0)) {
            setError('Please fill all item fields correctly');
            return;
        }

        setLoading(true);
        try {
            const transactionType = isRewash ? 'RE' : (isEdit ? (headerIsOut ? 'OUT' : 'IN') : type?.toUpperCase());
            const payload = {
                type: transactionType,
                shift,
                isInfected,
                dirtyWeightMorning: Number(dirtyWeightMorning),
                dirtyWeightAfternoon: Number(dirtyWeightAfternoon),
                cleanWeightMorning: Number(cleanWeightMorning),
                cleanWeightAfternoon: Number(cleanWeightAfternoon),
                linenStatus: weightStatus,
                companyId: headerIsOut ? companyId : null,
                externalStaffName: headerIsOut ? externalStaffName : null,
                departmentId: !headerIsOut ? departmentId : null,
                deptStaffName: !headerIsOut ? deptStaffName : null,
                parentId: parentId ? Number(parentId) : null,
                items: selectedItems
            };

            if (isEdit) {
                await api.put(`/transactions/${id}`, payload);
            } else {
                await api.post('/transactions', payload);
            }
            navigate('/tracking');
        } catch (err: any) {
            setError(err.response?.data?.message || 'Transaction failed');
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="min-h-screen bg-[#f8fafc] p-0 overflow-x-hidden">
            <div className="max-w-3xl mx-auto p-4 sm:p-6 lg:p-8">
                {/* Premium Header */}
                <header className="flex flex-col sm:flex-row items-start sm:items-center gap-3 sm:gap-4 mb-5 sm:mb-8 animate-in fade-in slide-in-from-top-4 duration-500">
                    <button
                        onClick={() => navigate('/')}
                        className="p-2 sm:p-3 bg-white rounded-xl sm:rounded-2xl shadow-sm text-gray-500 hover:text-indigo-600 hover:shadow-md transition-all duration-300"
                    >
                        <ArrowLeft size={18} />
                    </button>
                    <div>
                        <div className="flex flex-wrap items-center gap-2 mb-0.5 sm:mb-1">
                            <span className={`p-1 sm:p-1.5 rounded-lg text-white ${isOut ? 'bg-blue-600' : 'bg-emerald-600'}`}>
                                <Activity size={18} />
                            </span>
                            <h1 className="text-lg sm:text-2xl md:text-3xl font-[950] text-slate-900 tracking-tight">
                                {isEdit ? `แก้ไขรายการ ${docNo}` : (isRewash ? 'ส่งซักซ้ำ (Rewash)' : (isOut ? 'ส่งซักภายนอก' : 'ส่งซักภายใน'))}
                            </h1>
                        </div>
                        <p className="text-stone-800 text-xs sm:text-base font-bold tracking-tight">
                            {isEdit ? 'แก้ไขข้อมูลใบนำส่งที่ระบุ' : (isOut ? 'บันทึกรายการผ้าส่งซักกับบริษัทคู่สัญญา' : 'บันทึกรายการผ้าส่งซักจากหน่วยงานภายใน')}
                        </p>
                    </div>
                </header>

                <form onSubmit={handleSubmit} className="relative space-y-6 md:space-y-12 animate-in fade-in slide-in-from-bottom-4 duration-700">
                    {/* Sticky Summary Dashboard */}
                    <div className="sticky top-0 sm:top-2 z-[40] bg-white/95 backdrop-blur-md rounded-xl sm:rounded-2xl p-2 sm:p-3 border border-slate-100 shadow-[0_10px_30px_-10px_rgba(0,0,0,0.1)] flex flex-wrap items-center justify-between gap-1 sm:gap-4 px-3 sm:px-6 md:px-8">
                        <div className="flex flex-wrap items-center gap-2 sm:gap-5">
                            <div className="flex flex-col">
                                <span className="text-[8px] sm:text-[9px] font-black text-slate-500 uppercase tracking-widest leading-tight">ประเภท</span>
                                <span className={`font-black text-xs sm:text-sm ${isRewash ? 'text-amber-600' : (headerIsOut ? 'text-blue-600' : 'text-emerald-600')}`}>
                                    {isRewash ? 'ซักซ้ำ' : (headerIsOut ? 'ส่งภายนอก' : 'ส่งภายใน')}
                                </span>
                            </div>
                            <div className="hidden sm:block w-px h-6 bg-slate-100"></div>
                            <div className="flex flex-col">
                                <span className="text-[8px] sm:text-[9px] font-black text-slate-500 uppercase tracking-widest leading-tight">รายการ</span>
                                <span className="font-black text-slate-700 text-xs sm:text-sm">{totalItems} ชนิด</span>
                            </div>
                            <div className="hidden sm:block w-px h-6 bg-slate-100"></div>
                            <div className="flex flex-col">
                                <span className="text-[8px] sm:text-[9px] font-black text-slate-500 uppercase tracking-widest leading-tight">รวมจำนวน</span>
                                <span className="font-black text-indigo-600 text-base sm:text-lg">{totalQty} ชิ้น</span>
                            </div>
                        </div>
                        <div className="flex items-center gap-1 sm:gap-2">
                            <div className={`px-2 py-1 rounded-lg font-black text-[9px] sm:text-xs ${shift === 'MORNING' ? 'bg-blue-50 text-blue-600' : 'bg-orange-50 text-orange-600'}`}>
                                {shift === 'MORNING' ? 'AM' : 'PM'}
                            </div>
                        </div>
                    </div>

                    {error && (
                        <div className="bg-red-50 text-red-600 p-6 rounded-[2rem] font-bold text-sm border border-red-100 flex items-center gap-3 shadow-sm shadow-red-100">
                            <div className="bg-red-600 text-white p-1 rounded-full"><X size={14} /></div>
                            {error}
                        </div>
                    )}

                    {/* Section 1: Basic Info */}
                    <div className="bg-white rounded-2xl lg:rounded-3xl p-4 sm:p-6 lg:p-8 border border-slate-200 shadow-[0_10px_30px_-15px_rgba(0,0,0,0.05)] relative overflow-hidden group">
                        <div className="flex items-center gap-3 mb-4 lg:mb-6">
                            <div className="w-8 h-8 lg:w-10 lg:h-10 bg-slate-900 text-white rounded-xl flex items-center justify-center font-black text-base lg:text-lg shadow-lg shadow-slate-200">
                                01
                            </div>
                            <div>
                                <h3 className="text-lg lg:text-xl font-black text-slate-800 tracking-tight">ข้อมูลการทำรายการ</h3>
                                <p className="text-slate-800 text-[10px] lg:text-xs font-bold">หน่วยงานหรือบริษัทที่ดำเนินการ</p>
                            </div>
                        </div>
                        <div className="absolute top-0 right-0 p-8 text-slate-50 opacity-10 group-hover:opacity-20 transition-opacity pointer-events-none">
                            <ClipboardList size={120} />
                        </div>

                        <div className="relative grid grid-cols-1 md:grid-cols-1 gap-8">
                            {/* <div className="space-y-4">
                                <label className="text-[11px] font-[900] text-slate-800  uppercase tracking-widest ml-1 flex items-center gap-2">
                                    <Info size={14} /> สถานะผ้า
                                </label>
                                <div
                                    onClick={() => setIsInfected(!isInfected)}
                                    className={`flex items-center justify-between p-5 rounded-2xl border-2 transition-all duration-300 cursor-pointer ${isInfected
                                        ? 'bg-red-50 border-red-200 text-red-600 shadow-inner'
                                        : 'bg-slate-50 border-transparent text-slate-800     hover:border-slate-200'
                                        }`}
                                >
                                    <div className="flex items-center gap-3">
                                        <div className={`p-2 rounded-lg ${isInfected ? 'bg-red-600 text-white' : 'bg-slate-200 text-slate-800   '}`}>
                                            <Activity size={18} />
                                        </div>
                                        <span className="font-black text-lg">ผ้าติดเชื้อ (Infected)</span>
                                    </div>
                                    <div className={`w-6 h-6 rounded-full border-2 flex items-center justify-center transition-all ${isInfected ? 'bg-red-600 border-red-600 shadow-lg' : 'border-slate-300'
                                        }`}>
                                        {isInfected && <div className="w-2 h-2 bg-white rounded-full"></div>}
                                    </div>
                                </div>
                            </div> */}

                            <div className="space-y-2">
                                <label className="block text-[12px] font-[900] text-slate-800 uppercase tracking-widest ml-1">
                                    {headerIsOut ? 'บริษัทคู่สัญญา (Company)' : 'แผนกที่ส่ง (Department)'}
                                </label>
                                <div className="relative group/select">
                                    <select
                                        value={headerIsOut ? companyId : departmentId}
                                        onChange={e => headerIsOut ? setCompanyId(e.target.value) : setDepartmentId(e.target.value)}
                                        className="w-full bg-slate-50 border-2 border-transparent focus:border-indigo-500 focus:bg-white rounded-2xl p-4 outline-none transition-all duration-300 font-black text-slate-700 text-lg shadow-inner appearance-none cursor-pointer"
                                        required
                                    >
                                        <option value="">-- {headerIsOut ? 'เลือกบริษัท' : 'เลือกแผนก'} --</option>
                                        {headerIsOut
                                            ? companies.map(c => <option key={c.id} value={c.id}>{c.name}</option>)
                                            : departments.map(d => <option key={d.id} value={d.id}>{d.name}</option>)
                                        }
                                    </select>
                                    <div className="absolute right-4 top-1/2 -translate-y-1/2 pointer-events-none text-slate-800    ">
                                        <ArrowLeft className="-rotate-90" size={20} />
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>

                    {!isRewash && (
                        <>
                            {/* Section 2: Weights */}
                            <div className="bg-white rounded-3xl lg:rounded-[2.5rem] p-6 sm:p-8 lg:p-10 border border-slate-100 shadow-[0_15px_40px_-15px_rgba(0,0,0,0.05)] overflow-hidden">
                                <div className="flex items-center gap-4 mb-6 lg:mb-10">
                                    <div className="w-10 h-10 lg:w-12 lg:h-12 bg-blue-600 text-white rounded-2xl flex items-center justify-center font-black text-lg lg:text-xl shadow-lg shadow-blue-100">
                                        02
                                    </div>
                                    <div>
                                        <h3 className="text-xl lg:text-2xl font-black text-slate-800 tracking-tight">รายละเอียดผ้า</h3>
                                        <p className="text-slate-800 text-xs lg:text-sm font-bold">ระบุน้ำหนักรวมตามรอบเวลาที่กำหนด</p>
                                    </div>
                                </div>

                                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                                    <div className="space-y-4">
                                        <label className="block text-[11px] font-[900] text-slate-800 uppercase tracking-widest ml-1">รอบเวลา (Select Period)</label>
                                        <div className="flex gap-2 p-1.5 bg-slate-100 rounded-2xl">
                                            <button
                                                type="button"
                                                onClick={() => setShift('MORNING')}
                                                className={`flex-1 py-4 rounded-xl font-black transition-all duration-300 ${shift === 'MORNING' ? 'bg-white text-blue-600 shadow-md' : 'text-slate-800 hover:bg-slate-50'}`}
                                            >
                                                รอบเช้า (AM)
                                            </button>
                                            <button
                                                type="button"
                                                onClick={() => setShift('AFTERNOON')}
                                                className={`flex-1 py-4 rounded-xl font-black transition-all duration-300 ${shift === 'AFTERNOON' ? 'bg-white text-blue-600 shadow-md' : 'text-slate-800 hover:bg-slate-50'}`}
                                            >
                                                รอบบ่าย (PM)
                                            </button>
                                        </div>
                                    </div>

                                    {headerIsOut && (
                                        <div className="space-y-4">
                                            <label className="block text-[11px] font-[900] text-slate-800 uppercase tracking-widest ml-1">สถานะผ้า (Linen Status)</label>
                                            <div className="flex gap-2 p-1.5 bg-slate-100 rounded-2xl">
                                                <button
                                                    type="button"
                                                    onClick={() => setWeightStatus('DIRTY')}
                                                    className={`flex-1 py-4 rounded-xl font-black transition-all duration-300 ${weightStatus === 'DIRTY' ? 'bg-white text-rose-600 shadow-md' : 'text-slate-800 hover:bg-slate-50'}`}
                                                >
                                                    ผ้าสกปรก
                                                </button>
                                                <button
                                                    type="button"
                                                    onClick={() => setWeightStatus('CLEAN')}
                                                    className={`flex-1 py-4 rounded-xl font-black transition-all duration-300 ${weightStatus === 'CLEAN' ? 'bg-white text-emerald-600 shadow-md' : 'text-slate-800 hover:bg-slate-50'}`}
                                                >
                                                    ผ้าสะอาด
                                                </button>
                                            </div>
                                        </div>
                                    )}

                                    <div className="space-y-2">
                                        <label className="block text-[11px] font-[900] text-slate-800 uppercase tracking-widest ml-1">น้ำหนักรวม (Weight in KG)</label>
                                        <div className="relative">
                                            <input
                                                type="number"
                                                step="0.1"
                                                value={(() => {
                                                    if (weightStatus === 'DIRTY') {
                                                        return (shift === 'MORNING' ? dirtyWeightMorning : dirtyWeightAfternoon) || '';
                                                    } else {
                                                        return (shift === 'MORNING' ? cleanWeightMorning : cleanWeightAfternoon) || '';
                                                    }
                                                })()}
                                                onChange={e => {
                                                    const val = e.target.value === '' ? 0 : Number(e.target.value);
                                                    if (weightStatus === 'DIRTY') {
                                                        if (shift === 'MORNING') setDirtyWeightMorning(val);
                                                        else setDirtyWeightAfternoon(val);
                                                    } else {
                                                        if (shift === 'MORNING') setCleanWeightMorning(val);
                                                        else setCleanWeightAfternoon(val);
                                                    }
                                                }}
                                                className={`w-full bg-slate-50 border-2 border-transparent focus:bg-white rounded-2xl p-4 sm:p-6 md:p-8 outline-none transition-all duration-300 font-[950] text-3xl sm:text-4xl md:text-5xl shadow-inner text-right pr-20 md:pr-24 ${weightStatus === 'DIRTY' ? 'focus:border-rose-500 text-rose-600' : 'focus:border-emerald-500 text-emerald-600'}`}
                                                placeholder="0.0"
                                            />
                                            <span className="absolute right-6 top-1/2 -translate-y-1/2 font-black text-slate-300 text-xl tracking-tighter">KG</span>
                                        </div>
                                    </div>
                                </div>
                            </div>

                            {/* Section 3: Staff Info */}
                            <div className="bg-white rounded-3xl lg:rounded-[2.5rem] p-6 sm:p-8 lg:p-10 border border-slate-200 shadow-[0_15px_40px_-15px_rgba(0,0,0,0.05)] overflow-hidden">
                                <div className="flex items-center gap-4 mb-6 lg:mb-10">
                                    <div className="w-10 h-10 lg:w-12 lg:h-12 bg-orange-600 text-white rounded-2xl flex items-center justify-center font-black text-lg lg:text-xl shadow-lg shadow-orange-100 flex-shrink-0">
                                        03
                                    </div>
                                    <div className="flex items-center gap-3">
                                        <div>
                                            <h3 className="text-xl lg:text-2xl font-black text-slate-800 tracking-tight">ข้อมูลผู้ร่วมรายการ (Staff Info)</h3>
                                            <p className="text-slate-800     text-xs lg:text-sm font-bold">ชื่อพนักงานหรือเจ้าหน้าที่ผู้ดำเนินการ</p>
                                        </div>
                                    </div>
                                </div>
                                <div className="space-y-2">
                                    <label className="block text-[11px] font-[900] text-slate-800    uppercase tracking-widest ml-1">
                                        {headerIsOut ? 'ชื่อพนักงานบริษัท (External Staff)' : 'ชื่อเจ้าหน้าที่/พยาบาล (ผู้ส่งมอบ)'}
                                    </label>
                                    <input
                                        type="text"
                                        value={headerIsOut ? externalStaffName : deptStaffName}
                                        onChange={e => headerIsOut ? setExternalStaffName(e.target.value) : setDeptStaffName(e.target.value)}
                                        className="w-full bg-slate-50 border-2 border-transparent focus:border-indigo-500 focus:bg-white rounded-2xl p-4 outline-none transition-all duration-300 font-bold text-slate-700 text-lg shadow-inner"
                                        placeholder="ชื่อ-นามสกุล ของเจ้าหน้าที่..."
                                        required
                                    />
                                </div>
                            </div>
                        </>
                    )}

                    {/* Section 4: Items Table */}
                    <div className="bg-white rounded-2xl lg:rounded-3xl p-3 sm:p-4 md:p-6 border border-slate-100 shadow-[0_20px_40px_-20px_rgba(0,0,0,0.06)] min-h-[300px] md:min-h-[500px] flex flex-col overflow-hidden">
                        <div className="flex flex-col lg:flex-row justify-between items-start lg:items-center gap-3 mb-4 md:mb-6">
                            <div className="flex items-center gap-3">
                                <div className="w-8 h-8 md:w-10 md:h-10 bg-indigo-600 text-white rounded-xl flex items-center justify-center font-black text-base md:text-lg shadow-lg shadow-indigo-100">
                                    04
                                </div>
                                <div>
                                    <h3 className="text-lg md:text-xl font-black text-slate-800 tracking-tight">รายการผ้าทั้งหมด</h3>
                                    <p className="text-slate-800 text-[10px] md:text-xs font-bold">เลือกรายการและระบุจำนวน</p>
                                </div>
                            </div>
                            <button
                                type="button"
                                onClick={addItemRow}
                                className="w-full lg:w-auto bg-slate-900 text-white px-6 py-2 rounded-xl flex items-center justify-center gap-2 hover:bg-slate-800 active:scale-95 transition-all shadow-md font-black text-sm"
                            >
                                <Plus size={16} /> Add Item
                            </button>
                        </div>

                        <div className="flex-1 lg:-mx-10 overflow-x-auto custom-scrollbar">
                            <div className="min-w-[580px] lg:min-w-full lg:px-10">
                                <table className="w-full text-left border-separate border-spacing-y-3">
                                <thead>
                                    <tr className="text-[10px] font-black text-slate-800 uppercase tracking-wider italic">
                                        <th className="px-2 py-3 text-center w-8">#</th>
                                        <th className="px-4 py-3 text-left min-w-[150px] md:min-w-[240px]">รายการผ้า</th>
                                        <th className="px-2 py-3 text-center w-20">ประเภท/ไซส์</th>
                                        <th className="px-2 py-3 text-center w-10 text-red-500">ติดเชื้อ</th>
                                        <th className="px-3 py-3 text-center w-16">จำนวน</th>
                                        {isEdit && (
                                            <>
                                                <th className="px-3 py-3 text-center w-16 text-emerald-500">รับคืน</th>
                                                <th className="px-3 py-3 text-center w-16 text-rose-500">ค้างจ่าย</th>
                                                <th className="px-3 py-3 text-left text-slate-800">หมายเหตุ</th>
                                            </>
                                        )}
                                        <th className="px-2 py-3 w-16"></th>
                                    </tr>
                                </thead>
                                <tbody>
                                    {selectedItems.map((item, idx) => (
                                        <tr key={idx} className="group animate-in fade-in slide-in-from-bottom-2 duration-500" style={{ animationDelay: `${idx * 40}ms` }}>
                                            <td className="bg-white rounded-l-[2rem] p-3 text-center border-y-2 border-l-2 border-transparent group-hover:border-slate-100 transition-all duration-300 font-black text-slate-300">
                                                {idx + 1}
                                            </td>
                                            <td className="bg-white p-3 border-y-2 border-transparent group-hover:border-slate-100 group-hover:shadow-[0_10px_30px_-10px_rgba(0,0,0,0.05)] transition-all duration-300">
                                                <div className="flex flex-col gap-1">
                                                    <SearchableSelect
                                                        options={linenItems.map(li => ({
                                                            id: li.id,
                                                            name: li.name,
                                                            description: `${li.itemType || ''} ${li.itemSize || ''}`
                                                        }))}
                                                        value={item.itemId}
                                                        onChange={(val) => updateItem(idx, 'itemId', val)}
                                                        placeholder="เลือกรายการผ้า..."
                                                        required
                                                    />
                                                </div>
                                            </td>

                                            {/* Type/Size Column */}
                                            <td className="bg-white p-3 border-y-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                <div className="flex flex-col items-center justify-center">
                                                    {item.itemId ? (
                                                        <>
                                                            <span className="text-[9px] font-black text-slate-300 uppercase leading-none mb-1">
                                                                {(() => {
                                                                    const li = linenItems.find(it => it.id === Number(item.itemId));
                                                                    const mapping: Record<string, string> = { 'SHIRT': 'เสื้อ', 'PANTS': 'กางเกง', 'DOCTOR_SUIT': 'เสื้อสูทแพทย์', 'DRESS': 'ชุดเดรส/กระโปรง', 'SET': 'ชุดเข้าเซ็ท', 'OTHER': 'อื่นๆ' };
                                                                    return li?.itemType ? (mapping[li.itemType.toUpperCase()] || li.itemType) : '-';
                                                                })()}
                                                            </span>
                                                            <span className="text-blue-700 font-[950] text-sm leading-none bg-blue-50 px-2 py-1 rounded-lg">
                                                                {linenItems.find(it => it.id === Number(item.itemId))?.itemSize || '-'}
                                                            </span>
                                                        </>
                                                    ) : <span className="text-slate-200">-</span>}
                                                </div>
                                            </td>

                                            {/* Infected Column */}
                                            <td className="bg-white p-3 text-center border-y-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                    <button
                                                    type="button"
                                                    onClick={() => updateItem(idx, 'isInfected', !item.isInfected)}
                                                    className={`w-8 h-8 rounded-lg flex items-center justify-center transition-all duration-300 ${item.isInfected
                                                        ? 'bg-red-600 text-white shadow-lg shadow-red-100'
                                                        : 'bg-slate-50 text-slate-300 hover:text-slate-800 hover:bg-slate-100'
                                                        }`}
                                                    title="ติดเชื้อ"
                                                >
                                                    <Activity size={14} />
                                                </button>
                                            </td>

                                            {/* Quantity Column */}
                                            <td className="bg-white p-2 border-y border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                <div className="flex justify-center">
                                                    <input
                                                        type="number"
                                                        min="1"
                                                        value={item.sentQty || ''}
                                                        onChange={e => updateItem(idx, 'sentQty', e.target.value === '' ? 0 : Number(e.target.value))}
                                                        className="w-14 h-12 text-center bg-slate-50/50 rounded-2xl border-2 border-transparent focus:border-blue-500 focus:bg-white outline-none text-xl font-black text-blue-600 transition-all duration-300 shadow-inner"
                                                        required
                                                    />
                                                </div>
                                            </td>

                                            {/* Post-Transaction Columns (Only show in Edit) */}
                                            {isEdit && (
                                                <>
                                                    {/* Received Column */}
                                                    <td className="bg-white p-3 text-center border-y-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                        <span className="font-[950] text-emerald-500 text-lg">
                                                            {item.receivedQty || 0}
                                                        </span>
                                                    </td>

                                                    {/* Outstanding Column */}
                                                    <td className="bg-white p-3 text-center border-y-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                        <span className="font-[950] text-rose-500 text-lg">
                                                            {(item.sentQty || 0) - (item.receivedQty || 0)}
                                                        </span>
                                                    </td>

                                                    {/* Remark Column */}
                                                    <td className="bg-white p-3 border-y-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                        <input
                                                            type="text"
                                                            value={item.remark || ''}
                                                            onChange={e => updateItem(idx, 'remark', e.target.value)}
                                                            className="w-full bg-slate-50/50 border-2 border-transparent focus:border-indigo-500 focus:bg-white rounded-lg p-1.5 text-[10px] font-medium outline-none transition-all"
                                                            placeholder="ระบุหมายเหตุ..."
                                                        />
                                                    </td>
                                                </>
                                            )}

                                            {/* Actions Column */}
                                            <td className="bg-white rounded-r-[2rem] p-3 pr-8 border-y-2 border-r-2 border-transparent group-hover:border-slate-100 transition-all duration-300">
                                                <div className="flex items-center justify-end gap-1">
                                                    <button
                                                        type="button"
                                                        onClick={() => duplicateItem(idx)}
                                                        className="p-2 text-slate-200 hover:text-blue-500 hover:bg-blue-50 rounded-xl transition-all duration-300"
                                                        title="Duplicate"
                                                    >
                                                        <Copy size={18} />
                                                    </button>
                                                    {selectedItems.length > 1 && (
                                                        <button
                                                            type="button"
                                                            onClick={() => removeItemRow(idx)}
                                                            className="p-2 text-slate-200 hover:text-rose-500 hover:bg-rose-50 rounded-xl transition-all duration-300"
                                                            title="Delete"
                                                        >
                                                            <Trash2 size={18} />
                                                        </button>
                                                    )}
                                                </div>
                                            </td>
                                        </tr>
                                    ))}
                                </tbody>
                                </table>
                            </div>
                        </div>
                    </div>

                    {/* Footer Actions */}
                    <div className="flex flex-col md:flex-row gap-4 pt-4 sm:pt-10 pb-20">
                        <button
                            type="button"
                            onClick={() => navigate('/')}
                            className="flex-1 flex justify-center items-center gap-2 py-4 sm:py-5 border-2 border-slate-200 rounded-2xl font-black text-slate-800    hover:bg-slate-50 hover:text-slate-600 transition-all active:scale-95 text-base sm:text-lg"
                        >
                            <X size={24} /> ยกเลิก (Cancel)
                        </button>
                        <button
                            type="submit"
                            disabled={loading}
                            className={`relative flex-[2] flex justify-center items-center gap-2 py-4 sm:py-5 rounded-2xl font-black text-white text-lg sm:text-xl transition-all shadow-xl active:scale-95 disabled:opacity-50 ${headerIsOut
                                ? 'bg-blue-600 hover:bg-blue-700 shadow-blue-200'
                                : 'bg-emerald-600 hover:bg-emerald-700 shadow-emerald-200'
                                }`}
                        >
                            <Save size={24} /> {loading ? 'กำลังบันทึก...' : (isEdit ? 'บันทึกการแก้ไข' : 'บันทึกทำรายการ (Save)')}
                            {!loading && totalQty > 0 && (
                                <span className="absolute -top-2 -right-2 bg-white text-blue-600 rounded-full min-w-[1.5rem] h-6 px-1.5 flex items-center justify-center text-[10px] font-black shadow-lg border-2 border-current">
                                    {totalQty}
                                </span>
                            )}
                        </button>
                    </div>
                </form>

                {/* Floating Add Button (FAB) */}
                <button
                    type="button"
                    onClick={addItemRow}
                    className="fixed bottom-6 right-6 sm:bottom-8 sm:right-8 w-12 h-12 sm:w-16 sm:h-16 bg-slate-900 text-white rounded-full flex items-center justify-center shadow-2xl hover:bg-slate-800 hover:scale-110 active:scale-95 transition-all duration-300 z-50 group border-4 border-white"
                    title="เพิ่มรายการผ้า"
                >
                    <Plus size={24} className="sm:size-[28px] group-hover:rotate-90 transition-transform duration-500" />
                </button>
            </div>
        </div>
    );
};

export default TransactionForm;
