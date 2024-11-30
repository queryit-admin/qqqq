import { Handle, Position, NodeProps } from 'reactflow';
import { NodeData } from '@/types/flow';
import { useState } from 'react';
import NodeDescriptionModal from './NodeDescriptionModal';

const getNodeStyles = (type: string = 'default') => {
  const baseStyles = "px-4 py-3 shadow-lg rounded-lg border transition-all duration-200 ease-in-out transform hover:scale-105 cursor-pointer";
  
  switch (type) {
    case 'input':
      return `${baseStyles} bg-gradient-to-br from-blue-50 to-white border-blue-200`;
    case 'process':
      return `${baseStyles} bg-gradient-to-br from-purple-50 to-white border-purple-200`;
    case 'output':
      return `${baseStyles} bg-gradient-to-br from-green-50 to-white border-green-200`;
    default:
      return `${baseStyles} bg-gradient-to-br from-gray-50 to-white border-gray-200`;
  }
};

const getHandleStyles = (type: string = 'default') => {
  const baseStyles = "w-2 h-2 min-w-[8px] min-h-[8px] border-2 border-white transition-colors duration-200";
  
  switch (type) {
    case 'input':
      return `${baseStyles} !bg-blue-400 hover:!bg-blue-500`;
    case 'process':
      return `${baseStyles} !bg-purple-400 hover:!bg-purple-500`;
    case 'output':
      return `${baseStyles} !bg-green-400 hover:!bg-green-500`;
    default:
      return `${baseStyles} !bg-gray-400 hover:!bg-gray-500`;
  }
};

const getTypeStyles = (type: string = 'default') => {
  switch (type) {
    case 'input':
      return 'text-blue-600';
    case 'process':
      return 'text-purple-600';
    case 'output':
      return 'text-green-600';
    default:
      return 'text-gray-500';
  }
};

export default function CustomNode({ data, id }: NodeProps<NodeData>) {
  const [isModalOpen, setIsModalOpen] = useState(false);
  const nodeType = data.type || 'default';
  const inputCount = data.inputs?.length || 0;
  const outputCount = data.outputs?.length || 0;

  const handleNodeClick = () => {
    setIsModalOpen(true);
  };

  return (
    <>
      <div 
        className={getNodeStyles(nodeType)}
        onClick={handleNodeClick}
      >
        {/* Input Handles */}
        {inputCount > 0 && data.inputs?.map((handleId, index) => (
          <Handle
            key={handleId}
            type="target"
            position={Position.Left}
            id={handleId}
            className={getHandleStyles(nodeType)}
            style={{
              top: `${(index + 1) * (100 / (inputCount + 1))}%`,
            }}
          />
        ))}

        {/* Node Content */}
        <div className="flex flex-col items-center">
          <div className="text-sm font-medium">{data.label}</div>
          <div className={`text-xs ${getTypeStyles(nodeType)}`}>{nodeType}</div>
        </div>

        {/* Output Handles */}
        {outputCount > 0 && data.outputs?.map((handleId, index) => (
          <Handle
            key={handleId}
            type="source"
            position={Position.Right}
            id={handleId}
            className={getHandleStyles(nodeType)}
            style={{
              top: `${(index + 1) * (100 / (outputCount + 1))}%`,
            }}
          />
        ))}
      </div>

      {/* Description Modal */}
      <NodeDescriptionModal
        nodeId={id}
        isOpen={isModalOpen}
        onClose={() => setIsModalOpen(false)}
      />
    </>
  );
}
